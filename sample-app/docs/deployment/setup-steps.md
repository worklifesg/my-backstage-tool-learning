# Setup Steps

A step-by-step guide to reproduce this Backstage + ArgoCD integration from scratch.

## Prerequisites

- Docker v20+ installed
- Node.js v18+ with `corepack enable` (for Yarn)
- `kubectl` CLI
- `kind` CLI (v0.20+)
- A GitHub repo for the sample app manifests

## Step 1: Create a Kind Cluster

```yaml
# kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    extraPortMappings:
      - containerPort: 30080
        hostPort: 30080
      - containerPort: 30443
        hostPort: 30443
```

```bash
kind create cluster --name backstage-argocd --config kind-config.yaml
```

## Step 2: Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=argocd-server -n argocd --timeout=120s
```

Get the admin password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

Port-forward the ArgoCD API:

```bash
kubectl port-forward svc/argocd-server -n argocd 8443:443 &
```

## Step 3: Create the Sample App Manifests

Create `sample-app/app/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-app
  labels:
    app: sample-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: sample-app
  template:
    metadata:
      labels:
        app: sample-app
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
```

Create `sample-app/app/service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: sample-app
spec:
  selector:
    app: sample-app
  ports:
    - port: 80
      targetPort: 80
  type: ClusterIP
```

Push to GitHub.

## Step 4: Create the ArgoCD Application

```bash
argocd app create sample-app \
  --repo https://github.com/<your-user>/<your-repo>.git \
  --path sample-app/app \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default \
  --sync-policy automated \
  --auto-prune \
  --self-heal \
  --sync-option CreateNamespace=true
```

Verify:

```bash
argocd app get sample-app
# Should show: Status: Synced, Health: Healthy
```

## Step 5: Install Backstage ArgoCD Plugins

From the `backstage/` directory:

```bash
# Frontend plugin
yarn workspace app add @backstage-community/plugin-argocd

# Backend plugin
yarn workspace backend add @backstage-community/plugin-argocd-backend
```

## Step 6: Configure Backstage

See [Plugin Configuration](../integration/plugin-configuration.md) for detailed configuration of all files.

## Step 7: Register the Entity

Create `sample-app/catalog-info.yaml` with the `argocd/app-name` annotation (see [Files & Associations](../integration/files-and-associations.md)).

Add it to your catalog locations in `app-config.local.yaml`:

```yaml
catalog:
  locations:
    - type: file
      target: ../../../sample-app/catalog-info.yaml
```

## Step 8: Start Backstage

```bash
cd backstage
NODE_TLS_REJECT_UNAUTHORIZED=0 yarn start
```

> `NODE_TLS_REJECT_UNAUTHORIZED=0` is needed because ArgoCD uses a self-signed TLS certificate. **Never use this in production.**

## Step 9: Verify

1. Open `http://localhost:3000`
2. Navigate to **Catalog → sample-app**
3. Click the **ArgoCD** tab
4. You should see sync status, health, and deployment lifecycle
