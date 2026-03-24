# Troubleshooting

Common issues encountered during Backstage + ArgoCD integration and their fixes.

---

## ArgoCD Tab Not Appearing

**Symptom:** The entity page has no "ArgoCD" tab.

**Cause:** Missing or incorrect annotation.

**Fix:** Ensure `catalog-info.yaml` has:

```yaml
annotations:
  argocd/app-name: <exact-argocd-app-name>
```

The `isArgocdConfigured` condition checks for this annotation — no annotation, no tab.

Also verify the tab was added to the correct entity page in `EntityPage.tsx` (e.g., `serviceEntityPage` vs `defaultEntityPage`).

---

## Sync Status Shows "Unknown"

**Symptom:** The ArgoCD tab appears but shows sync status as "Unknown".

**Causes & Fixes:**

### 1. ArgoCD repo-server CrashLoopBackOff

Check pod status:

```bash
kubectl get pods -n argocd
```

If `argocd-repo-server` is in `CrashLoopBackOff`, it can't fetch Git data. Common on resource-constrained Kind clusters.

**Fix:** Wait for it to stabilize (Kubernetes backoff retry often resolves it), or increase resources:

```bash
kubectl patch deployment argocd-repo-server -n argocd --type=json \
  -p='[{"op":"replace","path":"/spec/template/spec/containers/0/resources/limits/memory","value":"512Mi"}]'
```

### 2. Revision metadata 503 errors

Check Backstage backend logs for:

```
503 Service Unavailable ... /api/v1/applications/sample-app/revisions/.../metadata
```

This is caused by repo-server being down — fix the repo-server first.

### 3. ArgoCD server not reachable

Verify port-forward is running:

```bash
curl -sk https://localhost:8443/api/version
```

If it fails, restart the port-forward:

```bash
kubectl port-forward svc/argocd-server -n argocd 8443:443 &
```

---

## "NotAllowedError" When Fetching Remote catalog-info.yaml

**Symptom:** Backstage logs show `NotAllowedError` when reading a URL-type catalog location.

**Cause:** Backstage blocks fetching from unknown hosts by default.

**Fix:** Add the host to `backend.reading.allow` in your config:

```yaml
backend:
  reading:
    allow:
      - host: raw.githubusercontent.com
```

---

## "Invalid content type" from ArgoCD API

**Symptom:** Backend logs show auth failures with ArgoCD.

**Cause:** ArgoCD API requires `Content-Type: application/json` for session creation.

**Fix:** This is handled automatically by the backend plugin. If you're testing manually:

```bash
curl -sk -H "Content-Type: application/json" \
  https://localhost:8443/api/v1/session \
  -d '{"username":"admin","password":"<password>"}'
```

---

## Wrong Plugin Exports (Compilation Errors)

**Symptom:** TypeScript errors like `Module has no exported member 'EntityArgocdContent'`.

**Cause:** Outdated tutorials reference exports that don't exist in newer plugin versions.

**Fix:** Check actual exports:

```bash
grep "^export" node_modules/@backstage-community/plugin-argocd/dist/index.d.ts
```

For `@backstage-community/plugin-argocd` v2.7.1, the correct exports are:

| Correct Export | Wrong (from old tutorials) |
|---------------|---------------------------|
| `ArgocdDeploymentSummary` | `EntityArgocdContent` |
| `ArgocdDeploymentLifecycle` | `EntityArgocdHistory` |
| `isArgocdConfigured` | `isArgocdAvailable` |

---

## app-config.local.yaml Overrides Catalog Locations

**Symptom:** Entities defined in `app-config.yaml` disappear when `app-config.local.yaml` exists.

**Cause:** `catalog.locations` is an array. When both config files define it, the local file **replaces** the base — it doesn't merge.

**Fix:** Include all needed locations in `app-config.local.yaml`, or only add unique locations there and keep shared ones in the base.

---

## TLS Certificate Errors

**Symptom:** `UNABLE_TO_VERIFY_LEAF_SIGNATURE` or `SELF_SIGNED_CERT_IN_CHAIN` errors.

**Cause:** ArgoCD uses a self-signed certificate.

**Fix (local dev only):**

```bash
NODE_TLS_REJECT_UNAUTHORIZED=0 yarn start
```

**Fix (production):** Use `NODE_EXTRA_CA_CERTS=/path/to/ca.crt` or configure proper certificates.

---

## Docker Too Old for Kind

**Symptom:** `kind create cluster` fails with `unknown flag: --cgroupns`.

**Cause:** Docker version < 20 doesn't support `--cgroupns` flag required by Kind v0.20+.

**Fix:** Upgrade Docker to v20+. On Ubuntu:

```bash
# Remove old Docker
sudo apt-get remove docker docker-engine docker.io containerd runc

# Install from Docker's official repo
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list
sudo apt-get update && sudo apt-get install docker-ce docker-ce-cli containerd.io
```

> On Ubuntu 20.04, you may also need to upgrade `libseccomp2` to 2.5.4+ from Debian repos.

---

## Quick Diagnostic Commands

```bash
# Check all ArgoCD pods
kubectl get pods -n argocd

# Check ArgoCD app status
argocd app get sample-app

# Test ArgoCD API connectivity
curl -sk https://localhost:8443/api/version

# Check Backstage backend logs (look for argocd errors)
# (visible in the terminal where you ran `yarn start`)

# Verify entity has correct annotation
kubectl get application sample-app -n argocd -o jsonpath='{.metadata.name}'
```
