# How ArgoCD-Backstage Integration Works

## The Core Idea

Backstage is a **developer portal** — it doesn't deploy anything. ArgoCD is a **GitOps CD tool** — it deploys but has no catalog. The integration connects them so developers can see their deployment status directly in the Backstage catalog without switching to the ArgoCD UI.

## How The Data Flows

```
                    ┌──────────────┐
                    │  Developer   │
                    │  Browser     │
                    └──────┬───────┘
                           │ opens entity page
                           ▼
                    ┌──────────────┐
                    │  Backstage   │
                    │  Frontend    │
                    │  (React)     │
                    │              │
                    │  ArgoCD      │
                    │  Plugin UI   │
                    └──────┬───────┘
                           │ API call to backend
                           ▼
                    ┌──────────────┐
                    │  Backstage   │
                    │  Backend     │
                    │  (Node.js)   │
                    │              │
                    │  ArgoCD      │
                    │  Backend     │──── reads `argocd/app-name`
                    │  Plugin      │     from entity annotations
                    └──────┬───────┘
                           │ authenticated HTTPS request
                           ▼
                    ┌──────────────┐
                    │  ArgoCD      │
                    │  Server API  │
                    │              │
                    │  /api/v1/    │
                    │  applications│
                    └──────────────┘
```

### Step-by-Step Flow

1. **User opens an entity page** in Backstage (e.g., `sample-app`)
2. **Frontend plugin** reads the entity's `argocd/app-name` annotation
3. **Frontend calls Backstage backend** (not ArgoCD directly — CORS/auth reasons)
4. **Backend plugin** looks up the ArgoCD instance from `appLocatorMethods` config
5. **Backend authenticates** with ArgoCD using the configured username/password → gets a JWT token
6. **Backend queries** ArgoCD REST API: `GET /api/v1/applications/{app-name}`
7. **ArgoCD returns** sync status, health, revision, resources, history
8. **Backend passes data** back to the frontend
9. **Frontend renders** the sync status, health badges, deployment lifecycle, and resource table

## The Key Link: Annotations

The **only thing** that connects a Backstage entity to an ArgoCD application is the annotation:

```yaml
metadata:
  annotations:
    argocd/app-name: sample-app    # ← must match the ArgoCD Application name exactly
```

This is a simple string match. The Backstage entity `sample-app` and the ArgoCD Application `sample-app` are completely independent objects — the annotation is the glue.

### What If Names Don't Match?

The entity name and the ArgoCD app name **don't need to be the same**. For example:

```yaml
metadata:
  name: my-frontend-service          # ← Backstage entity name
  annotations:
    argocd/app-name: frontend-prod   # ← ArgoCD application name (different!)
```

This works fine. The plugin uses the annotation value, not the entity name.

### Multiple ArgoCD Apps

You can associate multiple ArgoCD apps with a single entity:

```yaml
annotations:
  argocd/app-name: sample-app-dev, sample-app-staging, sample-app-prod
```

## Authentication Flow

```
Backstage Backend                          ArgoCD Server
      │                                         │
      │  POST /api/v1/session                    │
      │  {"username":"admin","password":"..."}   │
      │ ──────────────────────────────────────►  │
      │                                         │
      │  {"token": "eyJhbG..."}                  │
      │ ◄──────────────────────────────────────  │
      │                                         │
      │  GET /api/v1/applications/sample-app     │
      │  Authorization: Bearer eyJhbG...         │
      │ ──────────────────────────────────────►  │
      │                                         │
      │  { status: { sync: "Synced", ... } }     │
      │ ◄──────────────────────────────────────  │
```

The backend plugin handles token caching and renewal automatically.

## What Data Does Backstage Show?

| Field | Source | ArgoCD API Path |
|-------|--------|-----------------|
| Sync Status | `status.sync.status` | `/api/v1/applications/{name}` |
| Health Status | `status.health.status` | `/api/v1/applications/{name}` |
| Current Revision | `status.sync.revision` | `/api/v1/applications/{name}` |
| Revision Metadata | author, date, message | `/api/v1/applications/{name}/revisions/{rev}/metadata` |
| Resource Tree | pods, services, deployments | `/api/v1/applications/{name}/resource-tree` |
| Sync History | past sync operations | `status.history` |
