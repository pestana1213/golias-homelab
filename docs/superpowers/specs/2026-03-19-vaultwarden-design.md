# Vaultwarden Password Manager — Design Spec

**Date:** 2026-03-19
**Status:** Approved

## Overview

Add Vaultwarden (self-hosted Bitwarden-compatible password manager) to the homelab Kubernetes cluster, accessible at `http://password.golias.home`.

## Architecture

Multi-file layout with kustomization, following the immich pattern.

```
apps/vaultwarden/
  namespace.yaml
  secret.yaml
  deployment.yaml
  service.yaml
  ingress.yaml
  kustomization.yaml
```

Single container (`vaultwarden/server:latest`), SQLite database stored on a PVC at `/data`.

## Components

### namespace.yaml
Namespace: `vaultwarden`

### secret.yaml
Opaque secret `vaultwarden-secret` in namespace `vaultwarden`.
Key: `admin-token` — plaintext value (Vaultwarden hashes it internally at runtime).
Pattern matches `immich-secret`.

### deployment.yaml
- **PVC:** `vaultwarden-data-pvc`, `1Gi`, `ReadWriteOnce`, mounted at `/data`
- **Deployment:** `vaultwarden`, 1 replica
- **Image:** `vaultwarden/server:latest`
- **Port:** 80
- **Env vars:**
  - `ADMIN_TOKEN` — from `vaultwarden-secret` key `admin-token`
  - `DOMAIN` — `http://password.golias.home`
  - `SIGNUPS_ALLOWED` — `true` (disable via admin panel after first account created)
- **Resources:**
  - Requests: `128Mi` memory, `250m` CPU
  - Limits: `256Mi` memory, `500m` CPU
- No readiness/liveness probes (Vaultwarden starts quickly, keeping config simple)

### service.yaml
ClusterIP service on port 80, targeting container port 80.

### ingress.yaml
Host: `password.golias.home`
No annotations (matches immich ingress pattern).
Admin panel available at `http://password.golias.home/admin`.

### kustomization.yaml
Lists all resources: namespace, secret, deployment, service, ingress.

## Post-Deploy Steps

1. Navigate to `http://password.golias.home/admin` with the token from `secret.yaml`
2. Create your user account at `http://password.golias.home`
3. Disable signups via admin panel → General Settings → `Allow new signups: false`

## Out of Scope

- Shared PostgreSQL instance (separate future project)
- External/HTTPS access
- Backup automation
