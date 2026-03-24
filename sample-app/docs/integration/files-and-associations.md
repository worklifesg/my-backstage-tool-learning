# Files & Associations

Every file involved in the Backstage + ArgoCD integration, what it does, and how they connect.

## File Map

```
my-backstage-tool/
├── backstage/
│   ├── app-config.yaml                    ─── [1] Base Backstage config
│   ├── app-config.local.yaml              ─── [2] Local overrides (ArgoCD creds, catalog)
│   ├── packages/
│   │   ├── app/
│   │   │   └── src/components/catalog/
│   │   │       └── EntityPage.tsx         ─── [3] Frontend: ArgoCD tab rendering
│   │   └── backend/
│   │       └── src/
│   │           └── index.ts               ─── [4] Backend: ArgoCD plugin registration
│   └── examples/
│       ├── entities.yaml                  ─── Default example entities
│       └── auth-entity.yaml               ─── Auth-service demo component
│
├── sample-app/
│   ├── catalog-info.yaml                  ─── [5] Entity descriptor with ArgoCD annotation
│   ├── mkdocs.yml                         ─── [6] TechDocs configuration
│   ├── docs/                              ─── [7] Documentation (you're reading this!)
│   └── app/
│       ├── deployment.yaml                ─── [8] K8s Deployment manifest
│       └── service.yaml                   ─── [9] K8s Service manifest
│
└── kind-config.yaml                       ─── [10] Kind cluster configuration
```

## File Details

### [1] app-config.yaml — Base Configuration

**Purpose:** Default Backstage configuration shared across environments.

**ArgoCD-relevant sections:** None in the base config. ArgoCD config lives in the local override to keep credentials out of Git.

**Key concept:** Backstage merges `app-config.yaml` + `app-config.local.yaml`. The local file overrides or adds to the base. Array fields like `catalog.locations` are **replaced entirely**, not merged — so the local file must include all needed locations.

---

### [2] app-config.local.yaml — Local Overrides (GITIGNORED)

**Purpose:** Local development config containing sensitive data. **Never committed to Git** (covered by `*.local.yaml` in `.gitignore`).

**ArgoCD-specific config:**

```yaml
argocd:
  appLocatorMethods:
    - type: config
      instances:
        - name: local
          url: https://localhost:8443      # ArgoCD server URL
          username: admin                  # ArgoCD credentials
          password: <from-initial-secret>  # ArgoCD initial admin password
```

**Why this matters:**
- `instances[].name` — identifier used in logs; can be anything
- `instances[].url` — where the backend plugin sends API requests
- `instances[].username/password` — the backend plugin authenticates on each request

---

### [3] EntityPage.tsx — Frontend Plugin Wiring

**Purpose:** Defines what tabs and content appear on entity pages in the Backstage UI.

**What was added:**

```tsx
// Imports
import {
  ArgocdDeploymentLifecycle,
  ArgocdDeploymentSummary,
  isArgocdConfigured,
} from '@backstage-community/plugin-argocd';

// ArgoCD tab — only shows if the entity has argocd annotations
<EntityLayout.Route path="/argocd" title="ArgoCD" if={isArgocdConfigured}>
  <Grid container spacing={3}>
    <Grid item xs={12}>
      <ArgocdDeploymentSummary />     {/* Sync status, health, revision */}
    </Grid>
    <Grid item xs={12}>
      <ArgocdDeploymentLifecycle />   {/* Timeline of sync events */}
    </Grid>
  </Grid>
</EntityLayout.Route>
```

**Key exports from `@backstage-community/plugin-argocd` (v2.7.1):**

| Export | Type | Purpose |
|--------|------|---------|
| `ArgocdDeploymentSummary` | Component | Shows sync status, health, current revision |
| `ArgocdDeploymentLifecycle` | Component | Shows deployment history timeline |
| `isArgocdConfigured` | Condition | Returns true if entity has `argocd/app-name` annotation |

> **Gotcha:** Older tutorials may reference `EntityArgocdContent` or `isArgocdAvailable` — these don't exist in v2.7.1. Always check actual exports with `grep "^export" node_modules/@backstage-community/plugin-argocd/dist/index.d.ts`.

---

### [4] index.ts — Backend Plugin Registration

**Purpose:** Registers all backend plugins with the Backstage backend.

**What was added:**

```typescript
backend.add(import('@backstage-community/plugin-argocd-backend'));
```

This single line makes the backend plugin available. It automatically:
- Reads the `argocd` config from `app-config*.yaml`
- Creates API routes that the frontend plugin calls
- Handles authentication with ArgoCD

---

### [5] catalog-info.yaml — The Critical Link

**Purpose:** Describes the entity for the Backstage catalog and links it to ArgoCD.

```yaml
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: sample-app
  description: Sample Nginx app managed by Argo CD
  annotations:
    argocd/app-name: sample-app          # ← THE KEY: links to ArgoCD Application
    backstage.io/techdocs-ref: dir:.     # ← Enables TechDocs for this entity
```

**The `argocd/app-name` annotation is the ONLY connection** between a Backstage entity and an ArgoCD application. Without it, the ArgoCD tab won't appear.

---

### [6] mkdocs.yml — TechDocs Config

**Purpose:** Tells Backstage how to build this documentation site.

**Required by:** The `backstage.io/techdocs-ref: dir:.` annotation (which points to the directory containing `mkdocs.yml`).

---

### [7] docs/ — Documentation Source

Markdown files that MkDocs builds into the TechDocs site. The `nav` section in `mkdocs.yml` defines the structure.

---

### [8-9] deployment.yaml & service.yaml — K8s Manifests

**Purpose:** The actual application workloads that ArgoCD deploys.

**ArgoCD watches** the `sample-app/app/` path in the Git repo. Any changes to these files trigger a sync.

**These files have no Backstage-specific content** — they're pure Kubernetes manifests. The connection to Backstage is indirect: ArgoCD deploys them → Backstage queries ArgoCD about their status.

---

### [10] kind-config.yaml — Cluster Config

**Purpose:** Defines the Kind cluster with port mappings for NodePort services.

**Not Backstage-specific** — just infrastructure setup.

---

## Association Diagram

```
catalog-info.yaml                     ArgoCD Application
  annotations:                          name: sample-app
    argocd/app-name: sample-app  ═══►   source:
                                          path: sample-app/app/
                                          repoURL: github.com/...
                                        destination:
                                          server: kubernetes.default.svc

         ▲                                      │
         │ registered in                        │ deploys
         │                                      ▼

app-config.local.yaml                 Kind Cluster
  catalog:                              deployment/sample-app
    locations:                          service/sample-app
      - target: .../catalog-info.yaml
  argocd:
    instances:
      - url: https://localhost:8443  ═══► ArgoCD Server API
```
