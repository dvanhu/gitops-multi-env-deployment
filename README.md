# GitOps Multi-Environment Deployment

A **GitOps-based multi-environment deployment** project that manages application delivery across multiple environments (e.g., dev, staging, production) using Kubernetes and ArgoCD, with Git as the single source of truth for all environment configurations.

---

## What is GitOps?

GitOps is a deployment model where:
- The **desired state** of every environment is declared in Git
- An operator (ArgoCD) continuously **reconciles** the live cluster with that declared state
- All changes flow through **Git commits and PRs** — no direct `kubectl apply` in production

---

## Architecture Overview

```
Git Repository (this repo)
        │
        │  apps/
        │  ├── dev/
        │  ├── staging/
        │  └── production/
        │
        ▼
    ArgoCD
   (watches repo)
        │
   ┌────┴────────────────────┐
   ▼            ▼            ▼
dev namespace  staging ns   prod ns
(Kubernetes)  (Kubernetes) (Kubernetes)
```

ArgoCD monitors each directory and automatically syncs the corresponding Kubernetes namespace when changes are detected.

---

## Project Structure

```
gitops-multi-env-deployment/
└── apps/
    ├── dev/                    # Development environment manifests
    │   ├── deployment.yaml
    │   ├── service.yaml
    │   └── kustomization.yaml  # (if using Kustomize)
    ├── staging/                # Staging environment manifests
    │   ├── deployment.yaml
    │   ├── service.yaml
    │   └── kustomization.yaml
    └── production/             # Production environment manifests
        ├── deployment.yaml
        ├── service.yaml
        └── kustomization.yaml
```

---

## Tech Stack

| Tool | Role |
|---|---|
| Kubernetes | Container orchestration across environments |
| ArgoCD | GitOps operator — syncs Git state to cluster |
| Git / GitHub | Source of truth for all environment configs |
| Kustomize / Helm | Environment-specific manifest customization |

---

## Getting Started

### Prerequisites

- A running Kubernetes cluster
- `kubectl` configured
- ArgoCD installed on your cluster

### Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### Register Each Environment as an ArgoCD App

```bash
# Dev
argocd app create dev-app \
  --repo https://github.com/dvanhu/gitops-multi-env-deployment.git \
  --path apps/dev \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace dev \
  --sync-policy automated

# Staging
argocd app create staging-app \
  --repo https://github.com/dvanhu/gitops-multi-env-deployment.git \
  --path apps/staging \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace staging \
  --sync-policy automated

# Production
argocd app create prod-app \
  --repo https://github.com/dvanhu/gitops-multi-env-deployment.git \
  --path apps/production \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace production \
  --sync-policy automated
```

---

## Deployment Workflow

```
1. Developer opens a PR targeting the desired environment folder
         │
         ▼
2. PR is reviewed and merged into main
         │
         ▼
3. ArgoCD detects the change in Git
         │
         ▼
4. ArgoCD syncs the matching namespace on the cluster
         │
         ▼
5. Kubernetes performs a rolling update
```

This model ensures **no environment is ever modified directly** — all changes are auditable through Git history.

---

## Environment Strategy

| Environment | Branch/Path | Auto-Sync | Purpose |
|---|---|---|---|
| Dev | `apps/dev/` | Yes | Active development & testing |
| Staging | `apps/staging/` | Yes | Pre-production validation |
| Production | `apps/production/` | Manual gate recommended | Live traffic |

For production, it is recommended to require a manual sync approval in ArgoCD to prevent unintended rollouts.

---

## Key Concepts Demonstrated

- **Environment isolation** via separate Kubernetes namespaces
- **Declarative configuration** — desired state defined in YAML, not scripts
- **Automated reconciliation** — ArgoCD continuously enforces cluster state
- **Audit trail** — every deployment change is a traceable Git commit
- **Progressive delivery** — promote changes from dev → staging → production safely
