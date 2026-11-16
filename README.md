# linuxtips-gitops

A hands-on learning path for **GitOps with Argo CD on Google Kubernetes Engine (GKE)**, built one small step at a time.

## Related Repositories

| Repository | What it holds |
|---|---|
| **linuxtips-gitops** (this one) | the docs: one folder per module |
| **[linuxtips-gitops-gke](https://github.com/LuisGustavoBR/linuxtips-gitops-gke)** | Terraform and GitHub Actions that build the GKE clusters and install Argo CD |
| **[linuxtips-gitops-argocd](https://github.com/LuisGustavoBR/linuxtips-gitops-argocd)** | what Argo CD deploys: platform add-ons, demo apps and the files of each lesson |

## Table of Contents

Modules are added here as they are written.

### [Module 01 - GitOps Fundamentals and Argo CD Architecture](./01%20-%20GitOps%20Fundamentals%20and%20Argo%20CD%20Architecture/README.md)

- What GitOps is: the cluster state matches Git, and a merge is a deployment
- Where Argo CD fits in a branch-based workflow, and how automatic it should be
- The reconciliation loop: refresh (compare) and sync (apply), sync status and health
- Argo CD components and what each one does
- Anatomy of an Application: where (`destination`), what (`source`) and how (`syncPolicy`)

### [Module 02 - GKE Platform with Terraform and GitHub Actions](./02%20-%20GKE%20Platform%20with%20Terraform%20and%20GitHub%20Actions/README.md)

- Preparing the GCP project with `gcloud`: APIs, the Terraform state bucket, Workload Identity Federation and a least-privilege service account
- Creating the network: VPC, subnet with Pod and Service ranges, Cloud NAT, applied by a GitHub Actions pipeline with keyless login and approval gates
- Creating the GKE cluster: zonal Standard cluster with private nodes, DNS endpoint, Workload Identity, an on-demand system pool and a Spot apps pool; a reusable pipeline with static checks, change detection and a manual destroy workflow

## License

[MIT](./LICENSE)
