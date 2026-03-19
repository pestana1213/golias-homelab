# Tailscale Ingress — Design Spec

**Date:** 2026-03-19
**Status:** Approved

## Overview

Replace the existing `.golias.home` Traefik ingresses for Vaultwarden and Immich with Tailscale Ingress resources, using the Tailscale Kubernetes Operator. Each service gets its own node in the tailnet with valid HTTPS certificates.

**Resulting URLs:**
- Vaultwarden: `https://vaultwarden.tail654c62.ts.net`
- Immich: `https://immich.tail654c62.ts.net`

## Prerequisites (Manual — Already Done)

1. **HTTPS certificates enabled** in Tailscale admin console (DNS → Enable HTTPS Certificates)
2. **OAuth client created** in Tailscale admin (Settings → OAuth clients, scope: `devices`)
3. **Tailscale Kubernetes Operator installed** via Helm into the `tailscale` namespace:
   ```bash
   helm repo add tailscale https://pkgs.tailscale.com/helmcharts && helm repo update
   helm upgrade --install tailscale-operator tailscale/tailscale-operator --namespace=tailscale --create-namespace --set-string oauth.clientId="<ID>" --set-string oauth.clientSecret="<SECRET>" --wait
   ```

These steps are complete and do not need to be in the implementation plan.

## Architecture

The Tailscale Kubernetes Operator watches for Ingress resources with `ingressClassName: tailscale`. For each one, it provisions a Tailscale node (device) with a `tailscale.com/hostname` annotation as the device name, and obtains a Let's Encrypt HTTPS certificate via Tailscale's cert infrastructure. The TLS section in the Ingress spec signals the operator to enable HTTPS.

Services remain **ClusterIP** — no NodePort changes needed.

The existing Traefik ingresses are **replaced** (not kept alongside). Local `.golias.home` access is no longer needed.

## Changes

### Vaultwarden

**Files modified:**
- `apps/vaultwarden/ingress.yaml` — replace Traefik ingress with Tailscale ingress
- `apps/vaultwarden/deployment.yaml` — update `DOMAIN` env var

**`ingress.yaml` new content:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: vaultwarden-ingress
  namespace: vaultwarden
  annotations:
    tailscale.com/hostname: vaultwarden
spec:
  ingressClassName: tailscale
  tls:
    - hosts:
        - vaultwarden
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: vaultwarden
                port:
                  number: 80
```

**`deployment.yaml` change:** Update the `DOMAIN` env var value from `http://password.golias.home` to `https://vaultwarden.tail654c62.ts.net`.

### Immich

**Files modified:**
- `apps/immich/ingress.yaml` — replace Traefik ingress with Tailscale ingress

**`ingress.yaml` new content:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: immich-ingress
  namespace: immich
  annotations:
    tailscale.com/hostname: immich
spec:
  ingressClassName: tailscale
  tls:
    - hosts:
        - immich
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: immich-service
                port:
                  number: 2283
```

No other Immich files change.

## Out of Scope

- Other homelab services (adguard, homepage, monitoring, etc.) — not migrated in this spec
- Tailscale Funnel (public internet exposure)
- Keeping `.golias.home` DNS as fallback
