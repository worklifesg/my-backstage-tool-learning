# Deployment Overview

## What Is Deployed

**sample-app** is an Nginx-based web application deployed to a local Kubernetes cluster (Kind) and managed entirely through ArgoCD GitOps.

### Application Components

| Resource | Details |
|----------|---------|
| **Deployment** | `nginx:1.27`, 2 replicas |
| **Service** | ClusterIP on port 80 |
| **Namespace** | `default` |
| **ArgoCD Application** | `sample-app`, auto-sync enabled |

### Infrastructure

| Component | Details |
|-----------|---------|
| **Cluster** | Kind (`backstage-argocd`), single control-plane node |
| **ArgoCD** | v3.3.4, installed from stable manifests |
| **Backstage** | v1.48.0, running locally on ports 3000 (frontend) / 7007 (backend) |

## GitOps Flow

```
GitHub Repo (main branch)
    └── sample-app/
        ├── app/
        │   ├── deployment.yaml    ← K8s Deployment manifest
        │   └── service.yaml       ← K8s Service manifest
        └── catalog-info.yaml      ← Backstage entity descriptor

        ↓ ArgoCD watches this path

Kind Cluster (backstage-argocd)
    └── default namespace
        ├── deployment/sample-app   (2x nginx:1.27 pods)
        └── service/sample-app      (ClusterIP:80)

        ↓ Backstage queries ArgoCD API

Backstage UI
    └── Catalog → sample-app → ArgoCD tab
        ├── Sync Status
        ├── Health Status
        └── Deployment Lifecycle
```

## Auto-Sync Behavior

The ArgoCD Application is configured with:

- **`automated.prune: true`** — Deletes resources removed from Git
- **`automated.selfHeal: true`** — Reverts manual cluster changes to match Git
- **`syncOptions: CreateNamespace=true`** — Creates namespace if missing

This means any push to the `main` branch under `sample-app/app/` automatically triggers a sync to the cluster.
