# Module 2: GKE Platform with Terraform and GitHub Actions

## Overview

In this module we build the GKE platform that the rest of the learning path runs on.

From the second lesson on, everything is created by **Terraform**, applied by a **GitHub Actions** pipeline. But before Terraform can run in a pipeline, it needs two things that it can't create for itself:

1. **A bucket to store the Terraform state.** The state is the file where Terraform records what it has created. Pipeline runners are thrown away after each run, so the state can't live on them.
2. **A way for GitHub Actions to log in to Google Cloud**, without storing a password or a key file in GitHub.

So the first lesson prepares these by hand, with `gcloud` commands in the Cloud Shell. We do it once, and the pipeline takes over from there.

## Table of Contents

- [Lesson 1: Preparing the GCP Project](#lesson-1-preparing-the-gcp-project)
  - [1. Open the Cloud Shell and Set the Variables](#1-open-the-cloud-shell-and-set-the-variables)
  - [2. Enable the APIs](#2-enable-the-apis)
  - [3. Create the State Bucket](#3-create-the-state-bucket)
  - [4. Create the Workload Identity Pool and Provider](#4-create-the-workload-identity-pool-and-provider)
  - [5. Create the Terraform Service Account](#5-create-the-terraform-service-account)
  - [6. Allow GitHub Actions to Use the Service Account](#6-allow-github-actions-to-use-the-service-account)
  - [7. Configure GitHub](#7-configure-github)
  - [8. Check Everything](#8-check-everything)
- [Lesson 2: Creating the Network](#lesson-2-creating-the-network)
  - [1. What We Are Creating](#1-what-we-are-creating)
  - [2. The Terraform Files](#2-the-terraform-files)
  - [3. The Workflow](#3-the-workflow)
  - [4. Run It](#4-run-it)
  - [5. Check the Result](#5-check-the-result)
  - [6. If Something Fails](#6-if-something-fails)

---

# Lesson 1: Preparing the GCP Project

Here is what we are going to create:

```mermaid
flowchart LR
  gha[GitHub Actions<br/>job in environment lab] -- "1. OIDC token:<br/>I am repo X" --> wif[Workload Identity<br/>Federation]
  wif -- "2. short-lived<br/>credentials" --> sa[Service account<br/>terraform]
  sa -- "3. reads/writes" --> bucket[(Bucket<br/>Terraform state)]
  sa -- "4. creates" --> infra[Network, GKE...]
```

| Resource | Why |
|---|---|
| A **bucket** | stores the Terraform state |
| A **Workload Identity pool and provider** | lets GitHub Actions log in **without a key**. GitHub proves "I am a job of this repository", and Google returns credentials that expire in minutes |
| A **service account** `terraform` | the identity Terraform uses, with only the permissions it needs |

> **Note:** Why not a service account key (a JSON file stored as a GitHub secret)? A key never expires. Anyone who leaks it can use it from anywhere, until someone notices and deletes it. With Workload Identity Federation there is no key to leak.

## 1. Open the Cloud Shell and Set the Variables

Open the [Google Cloud console](https://console.cloud.google.com/), select your project and click the **Activate Cloud Shell** button (top right). The Cloud Shell is a terminal in the browser that already has `gcloud` installed and logged in with your account.

Set the variables used by every command in this lesson, changing `PROJECT_ID` to your project:

```bash
export PROJECT_ID="your-project-id"
export REGION="us-central1"
export GITHUB_OWNER="LuisGustavoBR"
export GITHUB_REPO="linuxtips-gitops-gke"

gcloud config set project "$PROJECT_ID"
export PROJECT_NUMBER=$(gcloud projects describe "$PROJECT_ID" --format='value(projectNumber)')
echo "$PROJECT_NUMBER"
```

We also need the **numeric IDs** of the GitHub owner and repository. Names can change, but these IDs never do:

```bash
export GITHUB_REPO_ID=$(curl -s "https://api.github.com/repos/${GITHUB_OWNER}/${GITHUB_REPO}" | jq -r .id)
export GITHUB_OWNER_ID=$(curl -s "https://api.github.com/repos/${GITHUB_OWNER}/${GITHUB_REPO}" | jq -r .owner.id)
echo "repo: $GITHUB_REPO_ID  owner: $GITHUB_OWNER_ID"
```

You should see two numbers. If you see `null`, check the owner and repository names (the repository must be public for this unauthenticated call).

> **Note:** If the Cloud Shell disconnects, these variables are lost. Run this block again before continuing.

## 2. Enable the APIs

A new project starts with almost every service switched off. We enable only what this module needs:

```bash
gcloud services enable \
  cloudresourcemanager.googleapis.com \
  compute.googleapis.com \
  container.googleapis.com \
  iam.googleapis.com \
  iamcredentials.googleapis.com \
  sts.googleapis.com \
  storage.googleapis.com
```

| API | Used for |
|---|---|
| `cloudresourcemanager` | reading and changing the project's IAM policy |
| `compute` | VPC, subnets, NAT, the nodes |
| `container` | GKE |
| `iam` | service accounts |
| `iamcredentials` | issuing short-lived credentials for a service account |
| `sts` | exchanging the GitHub token for Google credentials |
| `storage` | the state bucket |

It takes a minute or two. Expected output: `Operation "operations/..." finished successfully.`

## 3. Create the State Bucket

Bucket names are global across all of Google Cloud, so we prefix it with the project ID:

```bash
export BUCKET="${PROJECT_ID}-tfstate"

gcloud storage buckets create "gs://${BUCKET}" \
  --location="$REGION" \
  --uniform-bucket-level-access \
  --public-access-prevention

gcloud storage buckets update "gs://${BUCKET}" --versioning
```

| Option | Why |
|---|---|
| `--uniform-bucket-level-access` | permissions are managed only with IAM, on the whole bucket (no per-file permissions to forget about) |
| `--public-access-prevention` | the bucket can never be made public, even by mistake |
| `--versioning` | every change to the state keeps the previous version, so a broken state can be restored |

> **Note:** Terraform also needs a **lock**, so two runs never write the state at the same time. The GCS backend handles it inside the bucket itself, so there is nothing else to create.

## 4. Create the Workload Identity Pool and Provider

The **pool** is a container for external identities. The **provider** says which identity provider is trusted (GitHub) and which tokens are accepted.

```bash
gcloud iam workload-identity-pools create github \
  --location=global \
  --display-name="GitHub Actions"

gcloud iam workload-identity-pools providers create-oidc github-actions \
  --location=global \
  --workload-identity-pool=github \
  --display-name="GitHub Actions OIDC" \
  --issuer-uri="https://token.actions.githubusercontent.com" \
  --attribute-mapping="google.subject=assertion.sub,attribute.repository=assertion.repository,attribute.repository_id=assertion.repository_id" \
  --attribute-condition="assertion.repository_owner_id == '${GITHUB_OWNER_ID}' && assertion.repository_id == '${GITHUB_REPO_ID}'"
```

What the options mean:

- **`--issuer-uri`**: trust tokens signed by GitHub Actions.
- **`--attribute-mapping`**: which fields of the GitHub token Google keeps. The important one is `google.subject`, which for a job in an environment looks like `repo:LuisGustavoBR/linuxtips-gitops-gke:environment:lab`.
- **`--attribute-condition`**: the security gate. **Only tokens from this repository are accepted at all.** Without it, any GitHub repository could try to log in to your pool.

> **Note:** A deleted pool stays in a "deleted" state for 30 days, and its name can't be reused during that time. If you need to start over, undelete it (`gcloud iam workload-identity-pools undelete github --location=global`) instead of creating a new one with the same name.

## 5. Create the Terraform Service Account

```bash
gcloud iam service-accounts create terraform \
  --display-name="Terraform (GitHub Actions)"

export SA_EMAIL="terraform@${PROJECT_ID}.iam.gserviceaccount.com"
```

Now give it **only** the roles the next lessons need (network and GKE):

```bash
for role in \
  roles/compute.networkAdmin \
  roles/compute.securityAdmin \
  roles/container.admin \
  roles/iam.serviceAccountAdmin \
  roles/iam.serviceAccountUser \
  roles/serviceusage.serviceUsageConsumer
do
  gcloud projects add-iam-policy-binding "$PROJECT_ID" \
    --member="serviceAccount:${SA_EMAIL}" \
    --role="$role" \
    --condition=None \
    --quiet >/dev/null
  echo "granted $role"
done
```

| Role | Used for |
|---|---|
| `compute.networkAdmin` | VPC, subnets, router, NAT |
| `compute.securityAdmin` | firewall rules |
| `container.admin` | GKE clusters and node pools |
| `iam.serviceAccountAdmin` | creating the service account the GKE nodes run as |
| `iam.serviceAccountUser` | attaching that service account to the nodes |
| `serviceusage.serviceUsageConsumer` | calling APIs billed to this project |

One more permission is special. The GKE nodes run with their own service account, which needs a couple of roles in the project. To grant them, Terraform must be able to **edit the project's IAM policy**. That role (`projectIamAdmin`) is powerful: without limits, Terraform could grant itself anything, even Owner. So we attach an **IAM condition** that lists the only roles it is allowed to grant:

```bash
cat > iam-condition.yaml <<'EOF'
title: limited-grants
description: terraform may only grant the roles the GKE node service account needs
expression: "api.getAttribute('iam.googleapis.com/modifiedGrantsByRole', []).hasOnly(['roles/container.defaultNodeServiceAccount', 'roles/artifactregistry.reader'])"
EOF

gcloud projects add-iam-policy-binding "$PROJECT_ID" \
  --member="serviceAccount:${SA_EMAIL}" \
  --role="roles/resourcemanager.projectIamAdmin" \
  --condition-from-file=iam-condition.yaml \
  --quiet >/dev/null && echo "granted projectIamAdmin (limited)"
```

Finally, access to the state bucket:

```bash
gcloud storage buckets add-iam-policy-binding "gs://${BUCKET}" \
  --member="serviceAccount:${SA_EMAIL}" \
  --role="roles/storage.objectUser"
```

`objectUser` allows reading, writing and deleting files in this bucket only. The lock also needs it, because Terraform creates and deletes a lock file.

> **Note:** When a later lesson needs more permissions (Secret Manager, for example), we add that role here, in this guide, and say why. Starting small and growing on purpose is what **least privilege** means in practice.

## 6. Allow GitHub Actions to Use the Service Account

The last link: tokens from GitHub may **impersonate** (act as) the `terraform` service account, but only when the job runs in the GitHub Environment called `lab`:

```bash
gcloud iam service-accounts add-iam-policy-binding "$SA_EMAIL" \
  --role="roles/iam.workloadIdentityUser" \
  --member="principal://iam.googleapis.com/projects/${PROJECT_NUMBER}/locations/global/workloadIdentityPools/github/subject/repo:${GITHUB_OWNER}/${GITHUB_REPO}:environment:lab"
```

Why tie it to an environment? In GitHub, an **environment** can require a **reviewer**. So a job only gets these credentials after you approve the run. A pull request alone, or a commit pushed by someone else, can't change your infrastructure.

> **Note:** To keep things simple, every pipeline run that talks to Google Cloud (even a `terraform plan`) goes through the `lab` environment and needs your approval. Later we can split it into a read-only identity for plans and this one for applies.

## 7. Configure GitHub

Print the values the pipeline needs:

```bash
echo "GCP_PROJECT_ID=${PROJECT_ID}"
echo "GCP_WIF_PROVIDER=projects/${PROJECT_NUMBER}/locations/global/workloadIdentityPools/github/providers/github-actions"
echo "GCP_SERVICE_ACCOUNT=${SA_EMAIL}"
echo "TF_STATE_BUCKET=${BUCKET}"
```

In the `linuxtips-gitops-gke` repository on GitHub:

1. Go to **Settings → Secrets and variables → Actions → Variables** and create one **repository variable** for each line above (name on the left of `=`, value on the right). These values are not secret, because none of them grants access by itself, so they are variables, not secrets.
2. Go to **Settings → Environments → New environment** and create `lab`:
   - **Required reviewers**: add yourself
   - **Deployment branches and tags**: *Selected branches* → `main`

## 8. Check Everything

**The bucket.** Look for `versioning_enabled: true` and `public_access_prevention: enforced`:

```bash
gcloud storage buckets describe "gs://${BUCKET}"
```

**The provider.** It should be `ACTIVE`, with your repository ID in the condition:

```bash
gcloud iam workload-identity-pools providers describe github-actions \
  --location=global --workload-identity-pool=github \
  --format="value(state,attributeCondition)"
```

**The roles of the service account:**

```bash
gcloud projects get-iam-policy "$PROJECT_ID" \
  --flatten="bindings[].members" \
  --filter="bindings.members:serviceAccount:${SA_EMAIL}" \
  --format="table(bindings.role, bindings.condition.title)"
```

Expected output: the six roles, plus `projectIamAdmin` with the condition `limited-grants`.

**Who may impersonate it:**

```bash
gcloud iam service-accounts get-iam-policy "$SA_EMAIL"
```

Expected output: `roles/iam.workloadIdentityUser` for the `principal://.../environment:lab` member.

The real test, GitHub Actions actually logging in, happens in the next lesson, when the pipeline creates the network.

## Key Takeaways

- Terraform in a pipeline needs a remote state and a way to log in. Both are created once, by hand, before anything else.
- The state bucket has versioning (to restore a broken state) and can never be made public.
- Workload Identity Federation replaces key files: GitHub proves who it is, and Google returns short-lived credentials.
- The attribute condition makes the pool accept tokens from our repository only.
- The `terraform` service account gets only the roles it needs, and its IAM admin role is limited by a condition.
- Only jobs in the `lab` environment, which you approve, can use it.

---

# Lesson 2: Creating the Network

Now Terraform starts working, through the pipeline. The first thing we create is the network the GKE cluster will live in.

All the code is in the **[linuxtips-gitops-gke](https://github.com/LuisGustavoBR/linuxtips-gitops-gke)** repository:

```txt
# linuxtips-gitops-gke
.github/workflows/terraform.yaml   # the pipeline
network/                           # the network (a Terraform root module)
  versions.tf                      # Terraform/provider versions and the backend
  providers.tf                     # the Google provider
  variables.tf                     # declares the inputs (names, IP ranges)
  network.tf                       # the VPC and the subnet
  nat.tf                           # Cloud Router and Cloud NAT
  outputs.tf                       # what it prints at the end
  envs/
    lab.tfvars                     # the values of the lab environment
  .terraform.lock.hcl              # exact provider versions (generated, committed)
```

Each top-level folder (`network/` now, `gke/` next) is a separate Terraform project, with its own code and its own state. Keeping them apart keeps each one small, and a mistake in one can't touch the others.

## 1. What We Are Creating

```mermaid
flowchart TB
  internet((Internet))
  subgraph vpc["VPC gitops-vpc"]
    subgraph subnet["Subnet gitops-hub (us-central1)"]
      nodes["Nodes: 10.30.0.0/24"]
      pods["Pods (secondary): 10.30.32.0/19"]
      svcs["Services (secondary): 10.30.16.0/20"]
    end
    router[Cloud Router] --- nat[Cloud NAT]
  end
  nodes -- outbound only --> nat --> internet
```

| Resource | Why |
|---|---|
| **VPC** `gitops-vpc` | Our private network. In *custom mode*, Google creates no subnets by itself; we decide every range. |
| **Subnet** `gitops-hub` | Where the hub cluster lives. It has **three IP ranges** (see below). |
| **Cloud Router** + **Cloud NAT** | The nodes will have **no public IP**, so nothing on the internet can reach them directly. They still need to go *out* (pull images, clone GitHub repos), and Cloud NAT gives them that. |

**Why three IP ranges?** GKE clusters are *VPC-native*: Pods and Services get real IPs from the VPC, not from an overlay network hidden inside the cluster. So the subnet has:

| Range | CIDR | Size | Used for |
|---|---|---|---|
| primary | `10.30.0.0/24` | 256 IPs | one IP per **node** |
| `hub-pods` (secondary) | `10.30.32.0/19` | 8,192 IPs | **Pods**. GKE gives each node a block of Pod IPs. |
| `hub-services` (secondary) | `10.30.16.0/20` | 4,096 IPs | **Services** (ClusterIPs) |

The ranges must not overlap with each other, or with any network you may connect later.

## 2. The Terraform Files

**The backend.** It tells Terraform to keep the state in the bucket from Lesson 1:

```hcl
# network/versions.tf
terraform {
  required_version = "~> 1.17"

  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 8.11"
    }
  }

  # The state lives in the GCS bucket created by hand (docs: Module 2, Lesson 1).
  # bucket and prefix are passed by the pipeline with -backend-config,
  # so no project-specific value is committed here.
  backend "gcs" {}
}
```

The `backend "gcs" {}` block is **empty on purpose** (a *partial* backend). The bucket name is passed when the pipeline runs `terraform init`, so the code works for any project.

**The provider.** `project_id` has no default. The pipeline fills it from the GitHub variable `GCP_PROJECT_ID`, as the environment variable `TF_VAR_project_id` (Terraform reads any `TF_VAR_<name>` as the value of variable `<name>`):

```hcl
# network/providers.tf
provider "google" {
  project = var.project_id
  region  = var.region
}
```

**The VPC and the subnet:**

```hcl
# network/network.tf (comments removed)
resource "google_compute_network" "this" {
  name                    = "${var.name_prefix}-vpc"
  auto_create_subnetworks = false
  routing_mode            = "REGIONAL"
}

resource "google_compute_subnetwork" "hub" {
  name    = "${var.name_prefix}-hub"
  region  = var.region
  network = google_compute_network.this.id

  ip_cidr_range = var.nodes_cidr

  private_ip_google_access = true

  secondary_ip_range {
    range_name    = "hub-pods"
    ip_cidr_range = var.pods_cidr
  }

  secondary_ip_range {
    range_name    = "hub-services"
    ip_cidr_range = var.services_cidr
  }

  log_config {
    aggregation_interval = "INTERVAL_10_MIN"
    flow_sampling        = 0.1
    metadata             = "INCLUDE_ALL_METADATA"
  }
}
```

- `auto_create_subnetworks = false`: custom mode; only the subnets we declare exist.
- `private_ip_google_access = true`: nodes without public IPs can still reach Google APIs (Artifact Registry, Cloud Logging) over Google's internal network.
- `secondary_ip_range`: the Pod and Service ranges. The cluster will refer to them **by name** (`hub-pods`, `hub-services`).
- `log_config`: **VPC flow logs**, a record of the connections in and out of the subnet, useful to investigate network or security problems. Sampling 10% of the flows keeps the cost low.

**Cloud Router and Cloud NAT:**

```hcl
# network/nat.tf (excerpt)
resource "google_compute_router" "this" {
  name    = "${var.name_prefix}-router"
  region  = var.region
  network = google_compute_network.this.id
}

resource "google_compute_router_nat" "this" {
  name   = "${var.name_prefix}-nat"
  region = var.region
  router = google_compute_router.this.name

  nat_ip_allocate_option             = "AUTO_ONLY"
  source_subnetwork_ip_ranges_to_nat = "ALL_SUBNETWORKS_ALL_IP_RANGES"
}
```

- The **Cloud Router** is the regional piece Cloud NAT runs on.
- `AUTO_ONLY`: Google picks and manages the public IPs used for outbound traffic.
- `ALL_SUBNETWORKS_ALL_IP_RANGES`: nodes and Pods of every subnet can use the NAT.

**The values of each environment.** `variables.tf` only *declares* the inputs, with no defaults. The values live in one file per environment:

```hcl
# network/envs/lab.tfvars
region      = "us-central1"
name_prefix = "gitops"

nodes_cidr    = "10.30.0.0/24"  # 256 node IPs
services_cidr = "10.30.16.0/20" # 4,096 Service IPs
pods_cidr     = "10.30.32.0/19" # 8,192 Pod IPs
```

For now there is only `lab`. Adding another environment later (say `develop`) means a new `envs/develop.tfvars` with its own names and ranges, and the difference between the two is visible in one small file. The same code always runs everywhere; only the values change.

> **Note:** The repository's `.gitignore` ignores `*.tfvars` by default, because such files often hold secrets. We add an exception for `envs/*.tfvars`. They are committed on purpose, so **they must never contain a secret**. Even the project ID stays out: the pipeline passes it.

## 3. The Workflow

The pipeline has two jobs. Both run in the `lab` environment, so both wait for your approval:

```mermaid
flowchart LR
  push[push to main] --> plan["plan job<br/>(approve #1)"]
  plan -- "you read the plan<br/>in the job summary" --> apply["apply job<br/>(approve #2)"]
```

| Job | What it does |
|---|---|
| `plan` | logs in, runs `terraform init` and `terraform plan`, and writes the plan to the job **summary** |
| `apply` | after you read the plan and approve, logs in again and runs `terraform apply` |

The important parts:

```yaml
# .github/workflows/terraform.yaml (excerpt)
on:
  push:
    branches: [main]
    paths:
      - "network/**"
      - ".github/workflows/terraform.yaml"
  workflow_dispatch:
```

The pipeline runs on every push to `main` that changes Terraform code, or by hand (**Actions → terraform → Run workflow**). A push that only changes a README doesn't trigger it.

```yaml
# .github/workflows/terraform.yaml (excerpt)
permissions:
  contents: read

concurrency:
  group: terraform
  cancel-in-progress: false
```

- `permissions`: by default, jobs can only read the code.
- `concurrency`: two runs never happen at the same time, because they would fight over the same state. A new run waits for the previous one.

```yaml
# .github/workflows/terraform.yaml (excerpt)
  plan:
    runs-on: ubuntu-24.04
    environment: lab
    permissions:
      contents: read
      id-token: write
```

- `environment: lab` is what makes the job wait for your approval. It is also what Google checks: the service account only trusts jobs in `lab` (Lesson 1, step 6).
- `id-token: write` allows the job to ask GitHub for the OIDC token. Without it, there is nothing to exchange for Google credentials.

```yaml
# .github/workflows/terraform.yaml (excerpt)
      - name: Log in to Google Cloud (Workload Identity Federation, no keys)
        uses: google-github-actions/auth@v3.0.0
        with:
          workload_identity_provider: ${{ vars.GCP_WIF_PROVIDER }}
          service_account: ${{ vars.GCP_SERVICE_ACCOUNT }}
```

This is where the login happens: the action exchanges the GitHub token for short-lived credentials of the `terraform` service account. Every later step (and Terraform) uses them automatically.

> **Note:** Actions are pinned to an **exact version** (`@v3.0.0`), not to a floating major tag like `@v3`, which points to a new release every time one is published. Pinning means the pipeline only changes when we decide to update it. For even stricter security, you can pin to a full commit SHA: a tag can technically be moved by whoever controls the action's repository; a SHA can't.

```yaml
# .github/workflows/terraform.yaml
      - name: terraform init
        working-directory: ${{ env.TF_DIR }}
        run: |
          terraform init \
            -backend-config="bucket=${{ vars.TF_STATE_BUCKET }}" \
            -backend-config="prefix=${{ env.ENV }}/${{ env.TF_DIR }}"

      - name: terraform plan
        working-directory: ${{ env.TF_DIR }}
        run: |
          terraform plan -no-color -var-file="envs/${{ env.ENV }}.tfvars" | tee plan.txt
```

- `terraform init` completes the partial backend. The bucket comes from the GitHub variable, and the prefix is `<environment>/<folder>`, so the state goes to `gs://<bucket>/lab/network/`. Each environment and each Terraform folder gets its own state: `develop` will never overwrite `lab`, and `gke/` will never touch the state of `network/`.
- `-var-file` picks the values of the environment.
- `ENV` and `TF_DIR` are set once at the top of the workflow (`ENV: lab`, `TF_DIR: network`), so changing them in one place changes every path.

> **Note:** Why not write the bucket and prefix in `versions.tf`? Because the `backend` block **can't use variables**: Terraform reads it before any variable exists. That is why it stays empty and the values come from `-backend-config`.

> **Note:** Why does the `apply` job run a fresh `terraform apply` instead of reusing the plan from the first job? Passing a plan file between jobs means uploading it as an artifact. In a public repository anyone can download artifacts, and plan files can contain sensitive values. Since only you push to `main` and runs never overlap, a fresh apply right after you read the plan is safe enough here.

## 4. Run It

Commit and push the code:

```bash
git add .gitignore .github network
git commit -m "feat: add network and terraform pipeline"
git push
```

Then, on GitHub:

1. Open **Actions → terraform**. The run shows **Waiting**, because `plan` needs approval.
2. Click **Review deployments**, check `lab` and click **Approve and deploy**.
3. When `plan` finishes, open its **Summary** and read the plan. It should create four resources:

   ```txt
   Terraform will perform the following actions:

     # google_compute_network.this will be created
     # google_compute_router.this will be created
     # google_compute_router_nat.this will be created
     # google_compute_subnetwork.hub will be created
     ...
   Plan: 4 to add, 0 to change, 0 to destroy.
   ```

4. Now `apply` is waiting. Approve it the same way. It takes a minute or two and ends with:

   ```txt
   Apply complete! Resources: 4 added, 0 changed, 0 destroyed.

   Outputs:

   hub_secondary_ranges = {
     "pods" = "hub-pods"
     "services" = "hub-services"
   }
   hub_subnet = "gitops-hub"
   network = "gitops-vpc"
   ```

## 5. Check the Result

In the Cloud Shell (set `PROJECT_ID` again if the session is new):

```bash
gcloud compute networks list --filter="name=gitops-vpc"
gcloud compute networks subnets describe gitops-hub --region us-central1 \
  --format="yaml(ipCidrRange, privateIpGoogleAccess, secondaryIpRanges)"
gcloud compute routers nats list --router gitops-router --region us-central1
```

The subnet should show `10.30.0.0/24`, `privateIpGoogleAccess: true` and the two secondary ranges.

The state written by the pipeline is in the bucket:

```bash
gcloud storage ls "gs://${PROJECT_ID}-tfstate/lab/network/"
```

```txt
gs://your-project-id-tfstate/lab/network/default.tfstate
```

From now on, every change to `network/` reads and updates that file.

## 6. If Something Fails

Login problems break in the **"Log in to Google Cloud"** step or in `terraform init`, before Terraform even reads the code. They point to a step of Lesson 1:

| Error | Likely cause | Fix |
|---|---|---|
| `The given credential is rejected by the attribute condition` | the repository or owner ID in the provider condition is wrong | Lesson 1, steps 1 and 4: check `GITHUB_REPO_ID` / `GITHUB_OWNER_ID` |
| `Permission 'iam.serviceAccounts.getAccessToken' denied` | the job is not allowed to use the service account (wrong repo name or environment in the `principal://` member) | Lesson 1, step 6 |
| `Unable to retrieve ... id-token` / `ACTIONS_ID_TOKEN_REQUEST_URL` missing | the job lacks `id-token: write` | check the job's `permissions` |
| `storage.objects.list access ... denied` in `terraform init` | the service account can't use the bucket | Lesson 1, step 5 (bucket binding) |
| `Error: No value for required variable "project_id"` | the `GCP_PROJECT_ID` GitHub variable is missing or misspelled | Lesson 1, step 7 |
| Job stuck in **Waiting** | it is waiting for your approval | **Review deployments** |

Problems with the network itself show up later, in `plan` or `apply`:

| Error | Likely cause | Fix |
|---|---|---|
| `Compute Engine API has not been used in project ... or it is disabled` | the `compute` API is not enabled | Lesson 1, step 2 |
| `Required 'compute.networks.create' permission` | the service account lacks `compute.networkAdmin` | Lesson 1, step 5 |
| `Invalid IPCidrRange` / `... overlaps with ...` | a range in `envs/lab.tfvars` is invalid or overlaps another one | fix the CIDRs in `envs/lab.tfvars` |
| `No value for required variable "nodes_cidr"` (or another one) | the variable is missing from `envs/lab.tfvars` | add it to the file |

## Key Takeaways

- The platform is split into Terraform folders at the repository root, each with its own code and its own state; the first one is `network/`.
- GKE is VPC-native: the subnet has a primary range for nodes and secondary ranges for Pods and Services.
- Nodes will have no public IP; Cloud NAT gives them outbound internet access, and Private Google Access reaches Google APIs internally.
- A partial backend keeps project-specific values out of the code; the pipeline passes them to `terraform init`.
- `variables.tf` declares the inputs and `envs/<env>.tfvars` holds the values of each environment: same code everywhere, only the values change.
- `environment: lab` makes every job wait for your approval and is what lets it use the service account; `id-token: write` plus `google-github-actions/auth` is the whole keyless login.
