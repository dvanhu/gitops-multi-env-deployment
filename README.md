# gitops-multi-env-deployment

[![ArgoCD Sync](https://img.shields.io/badge/ArgoCD-Synced-brightgreen?logo=argo&logoColor=white)](https://argoproj.github.io/cd/)
[![Kustomize](https://img.shields.io/badge/Kustomize-Enabled-326CE5?logo=kubernetes&logoColor=white)](https://kustomize.io/)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-1.28+-326CE5?logo=kubernetes&logoColor=white)](https://kubernetes.io/)
[![GitOps](https://img.shields.io/badge/GitOps-Enabled-orange?logo=git&logoColor=white)](https://opengitops.dev/)

---

## Overview

This repository serves as the **GitOps configuration source of truth** for a multi-environment Kubernetes deployment platform. It contains all Kubernetes manifests structured using Kustomize, with a base configuration and environment-specific overlays for development, staging, and production.

ArgoCD continuously monitors this repository and reconciles the live cluster state with the declared configuration. No deployment is performed manually — all changes to cluster state are driven exclusively through commits to this repository.

This repository forms the **CD layer** of a two-repository GitOps architecture. The upstream CI layer is maintained in [`gitops-project`](https://github.com/your-org/gitops-project), which builds Docker images and updates the image tag in this repository.

---

## What is GitOps

GitOps is an operational framework that applies DevOps best practices — version control, collaboration, compliance, and CI/CD — to infrastructure automation. In a GitOps model:

- Git is the single source of truth for both application code and infrastructure configuration.
- All changes to deployed state are made through Git commits and pull requests, not through direct `kubectl` commands.
- An automated operator (in this case, ArgoCD) continuously reconciles the desired state declared in Git with the actual state running in the cluster.
- Drift between declared and live state is detected and corrected automatically.

This approach provides a complete, auditable history of every change to the infrastructure, simplifies rollbacks (revert a commit), and enforces separation of concerns between the CI and CD stages.

---

## Repository Structure

```
gitops-multi-env-deployment/
├── apps/
│   └── nginx/
│       ├── base/
│       │   ├── deployment.yaml         # Base Deployment manifest
│       │   ├── service.yaml            # Base Service manifest
│       │   └── kustomization.yaml      # Base Kustomize configuration
│       └── overlays/
│           ├── dev/
│           │   ├── kustomization.yaml  # Dev-specific patches and config
│           │   └── patch.yaml          # Dev replica count, resource limits
│           ├── staging/
│           │   ├── kustomization.yaml  # Staging-specific patches and config
│           │   └── patch.yaml          # Staging replica count, resource limits
│           └── prod/
│               ├── kustomization.yaml  # Production-specific patches and config
│               └── patch.yaml          # Production replica count, resource limits
└── README.md
```

---

## Kustomize Architecture

### Base Configuration

The `base/` directory contains the canonical Kubernetes manifests that are shared across all environments. These manifests define the core application structure without any environment-specific values.

```
apps/nginx/base/
├── deployment.yaml      # Defines the Deployment resource
├── service.yaml         # Defines the Service resource (ClusterIP)
└── kustomization.yaml   # Declares resources included in the base
```

The base `deployment.yaml` holds the image reference that is automatically updated by the CI pipeline on every successful build:

```yaml
# apps/nginx/base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-app
  template:
    metadata:
      labels:
        app: nginx-app
    spec:
      containers:
        - name: nginx-app
          image: your-dockerhub-username/nginx-app:abc1234   # Updated by CI
          ports:
            - containerPort: 80
```

The base `kustomization.yaml` declares which resources are part of the base:

```yaml
# apps/nginx/base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
```

### Overlay Configuration

Each environment overlay in `overlays/` extends the base by applying strategic merge patches or JSON patches. Overlays can modify replicas, resource requests and limits, environment variables, labels, and any other field without duplicating the entire manifest.

```yaml
# apps/nginx/overlays/prod/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
bases:
  - ../../base
patches:
  - path: patch.yaml
namePrefix: prod-
namespace: prod
```

```yaml
# apps/nginx/overlays/prod/patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-app
spec:
  replicas: 3
  template:
    spec:
      containers:
        - name: nginx-app
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 256Mi
```

### Environment Comparison

| Parameter         | dev       | staging   | prod      |
|-------------------|-----------|-----------|-----------|
| Namespace         | `dev`     | `staging` | `prod`    |
| Replicas          | 1         | 2         | 3         |
| CPU Request       | 50m       | 100m      | 100m      |
| CPU Limit         | 200m      | 400m      | 500m      |
| Memory Request    | 64Mi      | 128Mi     | 128Mi     |
| Memory Limit      | 128Mi     | 256Mi     | 256Mi     |
| Name Prefix       | `dev-`    | `staging-`| `prod-`   |

---

## Multi-Environment Architecture

The following diagram illustrates how the GitOps repository, Kustomize, ArgoCD, and Kubernetes namespaces interact to produce environment-specific deployments from a single source.

```
  gitops-multi-env-deployment (this repository)
  +----------------------------------------------------------+
  |                                                          |
  |  apps/nginx/base/          apps/nginx/overlays/          |
  |  +------------------+      +----------------------------+|
  |  | deployment.yaml  |<--+--| dev/kustomization.yaml     ||
  |  | service.yaml     |   +--| staging/kustomization.yaml ||
  |  | kustomization    |   +--| prod/kustomization.yaml    ||
  |  +------------------+      +----------------------------+|
  |                                                          |
  +----------------------------+-----------------------------+
                               |
                    ArgoCD watches repo
                    (polls every 3 min
                     or via webhook)
                               |
             +-----------------+-----------------+
             |                 |                 |
             v                 v                 v
    +------------------+ +------------------+ +------------------+
    | ArgoCD App: dev  | | ArgoCD App:      | | ArgoCD App: prod |
    | kustomize build  | | staging          | | kustomize build  |
    | overlays/dev     | | kustomize build  | | overlays/prod    |
    +--------+---------+ | overlays/staging | +--------+---------+
             |           +--------+---------+          |
             v                    v                    v
    +------------------+ +------------------+ +------------------+
    | Kubernetes        | | Kubernetes       | | Kubernetes       |
    | Namespace: dev    | | Namespace:       | | Namespace: prod  |
    |                   | | staging          | |                  |
    | dev-nginx-app     | | staging-nginx-   | | prod-nginx-app   |
    | Deployment: 1     | | app              | | Deployment: 3    |
    | replica           | | Deployment: 2    | | replicas         |
    +------------------+ | replicas         | +------------------+
                         +------------------+
```

---

## ArgoCD Integration

### How ArgoCD Monitors this Repository

ArgoCD is configured with one Application resource per environment. Each Application points to this repository and a specific overlay path. ArgoCD polls the repository (default interval: 3 minutes) or receives a webhook notification on push, compares the rendered Kustomize output against the live cluster state, and applies any detected differences.

### ArgoCD Application Manifest Reference

The following is a representative ArgoCD Application manifest for the production environment. Equivalent manifests exist for `dev` and `staging` with their respective overlay paths and namespaces.

```yaml
# argocd-app-prod.yaml (applied to the ArgoCD namespace in the cluster)
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nginx-app-prod
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/your-org/gitops-multi-env-deployment.git
    targetRevision: HEAD
    path: apps/nginx/overlays/prod
  destination:
    server: https://kubernetes.default.svc
    namespace: prod
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### Sync Policies

| Policy        | Description                                                                 |
|---------------|-----------------------------------------------------------------------------|
| `automated`   | ArgoCD applies changes automatically without manual approval                |
| `prune: true` | Resources removed from Git are also removed from the cluster                |
| `selfHeal`    | Drift from live state back to Git-declared state is corrected automatically |
| `CreateNamespace` | ArgoCD creates the target namespace if it does not already exist        |

---

## Environment Promotion Strategy

Promotion between environments is performed by updating the image tag in the base manifest and merging changes through a defined branch or pull request strategy. The recommended workflow is as follows.

```
  Code Commit (gitops-project repo)
           |
           v
  CI Pipeline Builds Image → Pushes to DockerHub
           |
           v
  CI updates apps/nginx/base/deployment.yaml
  (image tag: abc1234 → def5678)
           |
           v
  +--------+--------+
  |                 |
  v                 |
  ArgoCD detects   |
  change in base   |
  |                |
  v                |
  Deploys to DEV   |
  (automatic)      |
           |        |
           v        |
  QA validates on DEV
           |
           v
  Pull Request: promote to STAGING
  (merge or update overlay if needed)
           |
           v
  ArgoCD deploys to STAGING
  (automatic after merge)
           |
           v
  QA sign-off on STAGING
           |
           v
  Pull Request: promote to PROD
  (requires review and approval)
           |
           v
  ArgoCD deploys to PROD
  (automatic after merge, manual gate optional)
```

### Rollback Procedure

Because all configuration is stored in Git, rolling back to a previous state requires only reverting the relevant commit:

```bash
# Identify the commit to roll back to
git log --oneline apps/nginx/base/deployment.yaml

# Revert the most recent change
git revert HEAD
git push origin main

# ArgoCD detects the revert and redeploys the previous image tag automatically
```

---

## Setup Instructions

### Prerequisites

- A Kubernetes cluster (local: kind or minikube, or cloud: EKS, GKE, AKS)
- `kubectl` configured and pointing to the target cluster
- ArgoCD installed in the cluster
- Kustomize installed (version 4.x or later), or `kubectl` version 1.14+ (includes Kustomize)

### Step 1 — Fork or Clone the Repository

```bash
git clone https://github.com/your-org/gitops-multi-env-deployment.git
cd gitops-multi-env-deployment
```

### Step 2 — Install ArgoCD on the Cluster

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Wait for all ArgoCD pods to reach a Running state:

```bash
kubectl wait --for=condition=Ready pods --all -n argocd --timeout=120s
```

### Step 3 — Access the ArgoCD UI

```bash
# Port-forward the ArgoCD API server
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Retrieve the initial admin password
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath="{.data.password}" | base64 --decode
```

Open `https://localhost:8080` and log in with username `admin` and the retrieved password.

### Step 4 — Create the Kubernetes Namespaces

```bash
kubectl create namespace dev
kubectl create namespace staging
kubectl create namespace prod
```

### Step 5 — Register this Repository in ArgoCD

In the ArgoCD UI, navigate to **Settings > Repositories > Connect Repo** and provide this repository's URL. If the repository is private, supply a GitHub Personal Access Token or SSH key.

Alternatively, use the ArgoCD CLI:

```bash
argocd repo add https://github.com/your-org/gitops-multi-env-deployment.git \
  --username your-github-username \
  --password your-github-pat
```

### Step 6 — Apply the ArgoCD Application Manifests

```bash
# Apply all three environment applications
kubectl apply -f argocd-app-dev.yaml -n argocd
kubectl apply -f argocd-app-staging.yaml -n argocd
kubectl apply -f argocd-app-prod.yaml -n argocd
```

ArgoCD will immediately begin syncing the declared state to the cluster.

### Step 7 — Verify the Deployment

```bash
# Check ArgoCD application status
kubectl get applications -n argocd

# Check deployed resources per environment
kubectl get all -n dev
kubectl get all -n staging
kubectl get all -n prod
```

---

## Manual Kustomize Verification

To preview the rendered Kubernetes manifests for any environment without applying them:

```bash
# Preview dev environment output
kubectl kustomize apps/nginx/overlays/dev

# Preview staging environment output
kubectl kustomize apps/nginx/overlays/staging

# Preview prod environment output
kubectl kustomize apps/nginx/overlays/prod
```

To apply manually (not recommended in production — use ArgoCD instead):

```bash
kubectl apply -k apps/nginx/overlays/dev
```

---

## Related Repository

| Repository | Purpose |
|---|---|
| [`gitops-project`](https://github.com/dvanhu/gitops-project) | Application source code, Dockerfile, and GitHub Actions CI pipeline |
