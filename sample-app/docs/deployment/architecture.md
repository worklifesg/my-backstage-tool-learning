# Architecture

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────┐
│                     Developer Workflow                    │
│                                                          │
│   git push ──► GitHub Repo (main)                        │
│                    │                                     │
│                    ▼                                     │
│   ┌──────────────────────────────┐                       │
│   │        ArgoCD Server         │                       │
│   │   (Kind: argocd namespace)   │                       │
│   │                              │                       │
│   │  ● Watches: sample-app/app/  │                       │
│   │  ● Auto-sync: enabled        │                       │
│   │  ● Self-heal: enabled        │                       │
│   └──────────┬───────────────────┘                       │
│              │ deploys                                    │
│              ▼                                           │
│   ┌──────────────────────────────┐                       │
│   │     Kind Cluster             │                       │
│   │  (backstage-argocd)          │                       │
│   │                              │                       │
│   │  default namespace:          │                       │
│   │   ├── deploy/sample-app      │                       │
│   │   │   └── 2x nginx:1.27     │                       │
│   │   └── svc/sample-app        │                       │
│   │       └── ClusterIP:80      │                       │
│   └──────────────────────────────┘                       │
│              ▲                                           │
│              │ queries API                               │
│   ┌──────────┴───────────────────┐                       │
│   │     Backstage (localhost)     │                       │
│   │                              │                       │
│   │  :3000 ── Frontend (React)   │                       │
│   │  :7007 ── Backend (Node.js)  │                       │
│   │                              │                       │
│   │  ArgoCD Plugin ──► :8443     │                       │
│   │  (port-forwarded)            │                       │
│   └──────────────────────────────┘                       │
└─────────────────────────────────────────────────────────┘
```

## Component Relationships

### ArgoCD Side

| Component | Role |
|-----------|------|
| `argocd-server` | API server, UI — Backstage talks to this |
| `argocd-application-controller` | Reconciles desired state (Git) with actual state (cluster) |
| `argocd-repo-server` | Clones Git repos, generates manifests |
| `argocd-redis` | Caching layer for repo-server |
| `argocd-dex-server` | SSO/OIDC authentication |
| `argocd-applicationset-controller` | Manages ApplicationSets |
| `argocd-notifications-controller` | Sends notifications on sync events |

### Backstage Side

| Component | Role |
|-----------|------|
| `@backstage-community/plugin-argocd` | Frontend plugin — renders sync status, health, deployment lifecycle in the entity page |
| `@backstage-community/plugin-argocd-backend` | Backend proxy — authenticates with ArgoCD API, fetches application data |
| `catalog-info.yaml` | Links the Backstage entity to the ArgoCD application via `argocd/app-name` annotation |

## Network Topology (Local Dev)

```
localhost:3000  →  Backstage Frontend (webpack dev server)
                      │
                      ▼
localhost:7007  →  Backstage Backend (Node.js)
                      │
                      ▼ (ArgoCD backend plugin)
localhost:8443  →  kubectl port-forward → argocd-server:443
                      │
                      ▼
Kind cluster   →  argocd namespace (all ArgoCD components)
               →  default namespace (sample-app workloads)
```
