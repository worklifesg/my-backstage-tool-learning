# Sample App - Backstage + ArgoCD Integration

Welcome to the documentation for **sample-app** — a demo application showcasing end-to-end integration between **Backstage** and **Argo CD**.

## What This Demo Covers

| Area | Description |
|------|-------------|
| **Deployment** | Nginx app deployed to a Kind cluster via ArgoCD GitOps |
| **Backstage Catalog** | Component registered with ArgoCD annotations |
| **ArgoCD Plugin** | Frontend + backend plugins displaying sync/health status |
| **TechDocs** | This documentation, served via Backstage TechDocs |

## Quick Links

- [Deployment Overview](deployment/overview.md) — What's deployed and where
- [Architecture](deployment/architecture.md) — Component diagram
- [Setup Steps](deployment/setup-steps.md) — How to reproduce this setup
- [How ArgoCD-Backstage Integration Works](integration/how-it-works.md) — Core concepts
- [Files & Associations](integration/files-and-associations.md) — Every file involved and why
- [Plugin Configuration](integration/plugin-configuration.md) — Backend + frontend plugin wiring
- [Troubleshooting](integration/troubleshooting.md) — Common issues and fixes

## Tech Stack

| Component | Version |
|-----------|---------|
| Backstage | v1.48.0 |
| ArgoCD | v3.3.4 |
| Kubernetes | v1.34.0 (Kind) |
| Node.js | v24.14.0 |
| Docker | v28.1.1 |
