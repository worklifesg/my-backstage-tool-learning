# Plugin Configuration

Detailed reference for configuring the ArgoCD plugins in Backstage.

## Packages Required

Install from the `backstage/` directory:

```bash
# Frontend plugin — UI components for entity pages
yarn workspace app add @backstage-community/plugin-argocd

# Backend plugin — proxy that talks to ArgoCD API
yarn workspace backend add @backstage-community/plugin-argocd-backend
```

## Backend Configuration

### 1. Register the Backend Plugin

In `packages/backend/src/index.ts`, add:

```typescript
backend.add(import('@backstage-community/plugin-argocd-backend'));
```

That's it — the new Backstage backend system auto-discovers routes and config.

### 2. Configure ArgoCD Connection

In `app-config.local.yaml` (never in `app-config.yaml` — keep credentials out of Git):

```yaml
argocd:
  appLocatorMethods:
    - type: config
      instances:
        - name: local                        # Logical name (for logs/identification)
          url: https://localhost:8443         # ArgoCD API URL
          username: admin                    # ArgoCD username
          password: <your-argocd-password>   # ArgoCD password
```

### Configuration Options Reference

| Field | Required | Description |
|-------|----------|-------------|
| `type` | Yes | Must be `config` for static configuration |
| `instances[].name` | Yes | Identifier for this ArgoCD instance |
| `instances[].url` | Yes | Base URL of the ArgoCD API server |
| `instances[].username` | Yes* | Username for authentication |
| `instances[].password` | Yes* | Password for authentication |
| `instances[].token` | Yes* | Alternative: use a pre-generated API token instead of username/password |

*Either `username`+`password` or `token` is required.

### Multiple ArgoCD Instances

You can configure multiple ArgoCD instances:

```yaml
argocd:
  appLocatorMethods:
    - type: config
      instances:
        - name: dev
          url: https://argocd-dev.example.com
          token: ${ARGOCD_DEV_TOKEN}
        - name: prod
          url: https://argocd-prod.example.com
          token: ${ARGOCD_PROD_TOKEN}
```

The backend plugin will query **all instances** that match the entity's `argocd/app-name` annotation.

## Frontend Configuration

### 3. Update EntityPage.tsx

Add imports:

```tsx
import {
  ArgocdDeploymentLifecycle,
  ArgocdDeploymentSummary,
  isArgocdConfigured,
} from '@backstage-community/plugin-argocd';
```

Add the ArgoCD tab to your entity page (inside `serviceEntityPage` or wherever appropriate):

```tsx
<EntityLayout.Route path="/argocd" title="ArgoCD" if={isArgocdConfigured}>
  <Grid container spacing={3}>
    <Grid item xs={12}>
      <ArgocdDeploymentSummary />
    </Grid>
    <Grid item xs={12}>
      <ArgocdDeploymentLifecycle />
    </Grid>
  </Grid>
</EntityLayout.Route>
```

### Available Components (v2.7.1)

| Component | Description |
|-----------|-------------|
| `ArgocdDeploymentSummary` | Card showing sync status, health, revision, and last sync time |
| `ArgocdDeploymentLifecycle` | Timeline component showing deployment history |

### Conditional Rendering

`isArgocdConfigured` is a condition function that checks if the entity has the `argocd/app-name` annotation. The tab **only appears** for entities that have this annotation — other entities won't see the ArgoCD tab.

## Entity Annotation

### 4. Add the Annotation to catalog-info.yaml

```yaml
metadata:
  annotations:
    argocd/app-name: <argocd-application-name>
```

The value must **exactly match** the ArgoCD Application's `metadata.name`.

### Supported Annotations

| Annotation | Format | Example |
|------------|--------|---------|
| `argocd/app-name` | Single app | `sample-app` |
| `argocd/app-name` | Multiple apps (comma-separated) | `app-dev, app-staging, app-prod` |
| `argocd/project-name` | ArgoCD project filter | `my-project` |

## TLS / Self-Signed Certificates

If ArgoCD uses self-signed certs (common in local/dev), you need:

```bash
NODE_TLS_REJECT_UNAUTHORIZED=0 yarn start
```

> **Warning:** This disables TLS verification for ALL outgoing connections from the Backstage backend. Only use in local development.

For production, configure proper TLS certificates or use the `NODE_EXTRA_CA_CERTS` environment variable to trust your CA.

## Checklist

Use this to verify your setup is complete:

- [ ] `@backstage-community/plugin-argocd` installed in `packages/app`
- [ ] `@backstage-community/plugin-argocd-backend` installed in `packages/backend`
- [ ] Backend plugin registered in `packages/backend/src/index.ts`
- [ ] `argocd` config block in `app-config.local.yaml` with instance URL and credentials
- [ ] Entity page updated with ArgoCD tab in `EntityPage.tsx`
- [ ] Entity's `catalog-info.yaml` has `argocd/app-name` annotation
- [ ] ArgoCD server is accessible from the Backstage backend (port-forward or direct)
- [ ] Backstage started with `NODE_TLS_REJECT_UNAUTHORIZED=0` (if self-signed certs)
