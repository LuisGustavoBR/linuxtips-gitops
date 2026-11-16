# Module 3: Installing Argo CD and the First Application

## Overview

The GKE platform from Module 2 is ready. Now we install Argo CD on it and start using it.

In this module we will:

- install Argo CD with Terraform, through the same pipeline that created the cluster
- open the Argo CD UI and log in
- create our first Application (in the next lessons)

## Table of Contents

- [Lesson 1: Installing Argo CD with Terraform](#lesson-1-installing-argo-cd-with-terraform)
  - [1. Why Terraform Installs Argo CD](#1-why-terraform-installs-argo-cd)
  - [2. The Terraform Files](#2-the-terraform-files)
  - [3. The Pipeline](#3-the-pipeline)
  - [4. Run It](#4-run-it)
  - [5. Open the Argo CD UI](#5-open-the-argo-cd-ui)
  - [6. Troubleshooting](#6-troubleshooting)

---

# Lesson 1: Installing Argo CD with Terraform

## 1. Why Terraform Installs Argo CD

Argo CD deploys everything from Git, but it can't install itself: something has to put it in the cluster first. That something is Terraform, through the same pipeline that created the network and the cluster.

From then on, the split of responsibilities is:

| Who | Manages |
|---|---|
| **Terraform** (repo `linuxtips-gitops-gke`) | the platform: network, cluster **and Argo CD itself** |
| **Argo CD** (repo `linuxtips-gitops-argocd`) | everything that runs **on** the cluster: add-ons, applications |

Upgrading Argo CD is a commit that changes the chart version, reviewed and applied by the pipeline like any other infrastructure change.

The code is a new folder in **[linuxtips-gitops-gke](https://github.com/LuisGustavoBR/linuxtips-gitops-gke)**:

```txt
# linuxtips-gitops-gke
argocd/
  versions.tf          # Terraform, google and helm provider versions; backend "gcs" {}
  providers.tf         # how Terraform reaches the cluster
  variables.tf
  argocd.tf            # the Helm release
  values/argocd.yaml   # the chart values
  outputs.tf           # the installed chart version
  envs/lab.tfvars      # the values of the lab environment, including the chart version
```

## 2. The Terraform Files

**Reaching the cluster.** Terraform installs Argo CD with the **Helm** provider, which needs to talk to the Kubernetes API. We use the cluster's DNS endpoint (Module 2, Lesson 3) and the access token of whoever runs Terraform: in the pipeline, the `terraform` service account.

```hcl
# argocd/providers.tf (comments removed)
data "google_client_config" "this" {}

data "google_container_cluster" "hub" {
  name     = "${var.name_prefix}-hub"
  location = var.zone
}

provider "helm" {
  kubernetes = {
    host  = "https://${data.google_container_cluster.hub.control_plane_endpoints_config[0].dns_endpoint_config[0].endpoint}"
    token = data.google_client_config.this.access_token
  }
}
```

- `google_client_config` gives the current short-lived access token. No kubeconfig file and no key is involved.
- `google_container_cluster` (a `data` block) looks the cluster up by name, the same way `gke/` finds the network.
- The DNS endpoint uses a publicly trusted certificate, so there is no CA certificate to pass. IAM authorises the token: the `terraform` service account has `roles/container.admin`.

**The Helm release:**

```hcl
# argocd/argocd.tf (comments removed)
resource "helm_release" "argocd" {
  name             = "argocd"
  namespace        = "argocd"
  create_namespace = true

  repository = "https://argoproj.github.io/argo-helm"
  chart      = "argo-cd"
  version    = var.argocd_chart_version

  values = [file("${path.module}/values/argocd.yaml")]

  wait            = true
  timeout         = 600
  atomic          = true
  cleanup_on_fail = true
}
```

| Setting | What it does |
|---|---|
| `repository` + `chart` | the official chart from the Argo project (`argo-helm`) |
| `version` | the exact chart version, from `envs/lab.tfvars`. Never "latest", so nothing changes without a commit |
| `values` | our settings, kept in a separate YAML file that is easy to read and diff |
| `wait` + `timeout` | Terraform waits (up to 10 minutes) until every Argo CD Pod is ready |
| `atomic` + `cleanup_on_fail` | if the install fails, Helm rolls it back instead of leaving a half-installed Argo CD |

**The values:**

```yaml
# argocd/values/argocd.yaml (comments removed)
global:
  nodeSelector:
    node-pool: system

configs:
  params:
    server.insecure: true

dex:
  enabled: false
```

- **`global.nodeSelector`**: every Argo CD component runs on the on-demand `system` pool (label `node-pool: system`, set in Module 2), never on Spot nodes that can disappear.
- **`server.insecure: true`**: the Argo CD server speaks plain HTTP inside the cluster. For now we reach it with `kubectl port-forward`, which is already an encrypted tunnel. Later, TLS will end at the load balancer in front of it.
- **`dex.enabled: false`**: Dex is the SSO component (log in with GitHub, for example). It stays off until we set up SSO.

All the other options keep the chart defaults. The full list is in the chart's [values.yaml](https://github.com/argoproj/argo-helm/blob/main/charts/argo-cd/values.yaml).

**The chart version, per environment:**

```hcl
# argocd/envs/lab.tfvars
region      = "us-central1"
zone        = "us-central1-a"
name_prefix = "gitops"

argocd_chart_version = "10.18.0" # Argo CD v3.6.1
```

The version lives in the environment file on purpose. With more than one environment, an upgrade can be tested in one of them before the others.

## 3. The Pipeline

`argocd` is one more job in `terraform.yaml`, using the same reusable workflow, and it runs after `gke`:

```yaml
# .github/workflows/terraform.yaml (excerpt)
  argocd:
    needs: [checks, changes, gke]
    if: ${{ !failure() && !cancelled() && needs.changes.outputs.argocd == 'true' }}
    uses: ./.github/workflows/_terraform.yaml
    with:
      dir: argocd
    permissions:
      contents: read
      id-token: write
```

The `checks` and `changes` jobs also gained `argocd` in their folder lists.

The destroy workflow now removes Argo CD **before** the cluster, the reverse of the creation order:

```yaml
# .github/workflows/destroy.yaml (excerpt)
jobs:
  argocd:
    uses: ./.github/workflows/_terraform.yaml
    with:
      dir: argocd
      destroy: true
    ...
  gke:
    needs: argocd
    uses: ./.github/workflows/_terraform.yaml
    with:
      dir: gke
      destroy: true
```

## 4. Run It

The cluster must be running. If you destroyed it, run **Actions → terraform → Run workflow** first.

```bash
git add .github argocd
git commit -m "feat: install argo cd with terraform"
git push
```

This push changes the workflows, so every folder runs. `network` and `gke` plan `No changes`; approve them to move on. Then approve **argocd / plan** and read the summary. There is one resource to create:

```txt
  # helm_release.argocd will be created
  + resource "helm_release" "argocd" {
      + chart            = "argo-cd"
      + name             = "argocd"
      + namespace        = "argocd"
      + repository       = "https://argoproj.github.io/argo-helm"
      + version          = "10.18.0"
      ...
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```

Approve **argocd / apply**. It takes two or three minutes, while Terraform waits for the Pods. It ends with:

```txt
Apply complete! Resources: 1 added, 0 changed, 0 destroyed.

Outputs:

chart_version = "10.18.0"
```

## 5. Open the Argo CD UI

In the Cloud Shell, connect `kubectl` to the cluster (if you haven't yet) and look at the Argo CD Pods:

```bash
gcloud container clusters get-credentials gitops-hub \
  --location us-central1-a --project "$PROJECT_ID" --dns-endpoint

kubectl get pods -n argocd -o wide
```

```txt
NAME                                                READY   STATUS    NODE
argocd-application-controller-0                     1/1     Running   gke-gitops-hub-system-...
argocd-applicationset-controller-7c9f8d6b5-x2k8p    1/1     Running   gke-gitops-hub-system-...
argocd-notifications-controller-5d8b7c4f9-q7lzn     1/1     Running   gke-gitops-hub-system-...
argocd-redis-6b7d9f8c5-m4w2t                        1/1     Running   gke-gitops-hub-system-...
argocd-repo-server-84f6c7d9b-tr5vk                  1/1     Running   gke-gitops-hub-system-...
argocd-server-6f9b8c7d4-hn3jx                       1/1     Running   gke-gitops-hub-system-...
```

Every Pod runs on the `system` node. These are the components from [Module 1](../01%20-%20GitOps%20Fundamentals%20and%20Argo%20CD%20Architecture/README.md#3-the-components), without Dex, which is off for now.

**Get the admin password.** Argo CD generates a random password at install time and stores it in a Secret. It is base64-encoded, like every Secret value:

```bash
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath='{.data.password}' | base64 -d; echo
```

**Open the UI.** Create a tunnel from the Cloud Shell to the Argo CD server:

```bash
kubectl port-forward -n argocd svc/argocd-server 8080:80
```

Then click **Web Preview** (the eye icon at the top of the Cloud Shell) → **Preview on port 8080**. The Argo CD login page opens in a new tab. Log in with user `admin` and the password from above.

The tunnel lives as long as the command runs. Press `Ctrl+C` to close it.

> **Note:** On your own machine, the same `port-forward` works after `get-credentials`; then open `http://localhost:8080`.

The Applications page is empty for now: there is nothing in Git for Argo CD to deploy yet. That is the next lesson.

## 6. Troubleshooting

| Error | Likely cause | Fix |
|---|---|---|
| `argocd / plan`: `Error reading Container Cluster` / `not found` | the cluster is not running (it was destroyed) | run **Actions → terraform → Run workflow** to recreate it first |
| `Kubernetes cluster unreachable` / `403 Forbidden` | the `terraform` service account can't use the Kubernetes API | Module 2, Lesson 1, step 5: `roles/container.admin` |
| `context deadline exceeded` / timeout while installing | a Pod didn't become ready in 10 minutes (for example, the `system` node is not ready) | `kubectl get pods -n argocd` and `kubectl describe pod ...` to see why |
| Pods stuck in `Pending` with `node(s) didn't match Pod's node affinity/selector` | no node has the `node-pool: system` label | check the `system` pool in `gke/node_pools.tf` |
| Web Preview shows an error | the `port-forward` is not running, or it uses another port | run the `port-forward` command again, on port 8080 |

## Key Takeaways

- Argo CD can't install itself: Terraform installs it, through the same pipeline as the rest of the platform.
- Terraform reaches the cluster through its DNS endpoint with a short-lived token. No kubeconfig and no keys.
- The chart version is pinned per environment, so an upgrade is a reviewed commit.
- Every Argo CD component runs on the on-demand `system` pool.
- The initial admin password is in the `argocd-initial-admin-secret` Secret, and the UI is reachable with `kubectl port-forward` and the Cloud Shell Web Preview.
