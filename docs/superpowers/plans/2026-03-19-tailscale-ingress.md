# Tailscale Ingress Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the Traefik `.golias.home` ingresses for Vaultwarden and Immich with Tailscale Ingress resources, making them accessible at `https://vaultwarden.tail654c62.ts.net` and `https://immich.tail654c62.ts.net`.

**Architecture:** The Tailscale Kubernetes Operator (already installed) watches for Ingress resources with `ingressClassName: tailscale`. Each such Ingress creates a Tailscale device with the name from the `tailscale.com/hostname` annotation, with HTTPS handled automatically via the `tls` section. Services stay as ClusterIP.

**Tech Stack:** Kubernetes, Tailscale Kubernetes Operator, Kustomize.

---

## File Map

| File | Action | Change |
|------|--------|--------|
| `apps/vaultwarden/ingress.yaml` | Modify | Replace Traefik ingress with Tailscale ingress |
| `apps/vaultwarden/deployment.yaml` | Modify | Update `DOMAIN` env var to Tailscale URL |
| `apps/immich/ingress.yaml` | Modify | Replace Traefik ingress with Tailscale ingress |

---

### Task 1: Update Vaultwarden ingress

**Files:**
- Modify: `apps/vaultwarden/ingress.yaml`

- [ ] **Step 1: Replace the file content**

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

Key changes from the old file:
- Added `annotations: tailscale.com/hostname: vaultwarden`
- Changed `ingressClassName` to `tailscale`
- Added `tls` section (this tells the operator to provision HTTPS)
- Removed `spec.rules[0].host: password.golias.home`

- [ ] **Step 2: Validate**

```bash
kubectl apply --dry-run=client -f apps/vaultwarden/ingress.yaml
```

Expected: `ingress.networking.k8s.io/vaultwarden-ingress configured (dry run)`

- [ ] **Step 3: Commit**

```bash
git add apps/vaultwarden/ingress.yaml
git commit -m "feat(vaultwarden): migrate ingress to Tailscale"
```

---

### Task 2: Update Vaultwarden DOMAIN env var

**Files:**
- Modify: `apps/vaultwarden/deployment.yaml` (line 42)

The `DOMAIN` env var tells Vaultwarden what its public base URL is. It's used in invitation links and WebAuthn. It must match the actual URL users access the service from.

- [ ] **Step 1: Update the DOMAIN value**

In `apps/vaultwarden/deployment.yaml`, change line 42 from:
```yaml
            - name: DOMAIN
              value: http://password.golias.home
```
to:
```yaml
            - name: DOMAIN
              value: https://vaultwarden.tail654c62.ts.net
```

Everything else in the file stays unchanged.

- [ ] **Step 2: Validate**

```bash
kubectl apply --dry-run=client -f apps/vaultwarden/deployment.yaml
```

Expected:
```
persistentvolumeclaim/vaultwarden-data-pvc configured (dry run)
deployment.apps/vaultwarden configured (dry run)
```

- [ ] **Step 3: Commit**

```bash
git add apps/vaultwarden/deployment.yaml
git commit -m "feat(vaultwarden): update DOMAIN to Tailscale URL"
```

---

### Task 3: Update Immich ingress

**Files:**
- Modify: `apps/immich/ingress.yaml`

- [ ] **Step 1: Replace the file content**

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

Key changes from the old file:
- Added `annotations: tailscale.com/hostname: immich`
- Added `ingressClassName: tailscale`
- Added `tls` section
- Removed `spec.rules[0].host: immich.golias.home`

- [ ] **Step 2: Validate**

```bash
kubectl apply --dry-run=client -f apps/immich/ingress.yaml
```

Expected: `ingress.networking.k8s.io/immich-ingress configured (dry run)`

- [ ] **Step 3: Commit**

```bash
git add apps/immich/ingress.yaml
git commit -m "feat(immich): migrate ingress to Tailscale"
```

---

### Task 4: Apply to cluster and verify

- [ ] **Step 1: Apply Vaultwarden changes**

```bash
kubectl apply -k apps/vaultwarden/
```

Expected:
```
namespace/vaultwarden unchanged
secret/vaultwarden-secret unchanged
persistentvolumeclaim/vaultwarden-data-pvc unchanged
deployment.apps/vaultwarden configured
service/vaultwarden unchanged
ingress.networking.k8s.io/vaultwarden-ingress configured
```

- [ ] **Step 2: Apply Immich changes**

```bash
kubectl apply -k apps/immich/
```

Expected:
```
namespace/immich unchanged
...
ingress.networking.k8s.io/immich-ingress configured
```

- [ ] **Step 3: Wait for Tailscale devices to appear**

```bash
kubectl get ingress -n vaultwarden
kubectl get ingress -n immich
```

Wait until the `ADDRESS` column shows a Tailscale IP (100.x.x.x) for both — this means the operator has provisioned the devices. May take 30–60 seconds.

- [ ] **Step 4: Verify in Tailscale admin**

Go to [login.tailscale.com/admin/machines](https://login.tailscale.com/admin/machines) — you should see two new devices: `vaultwarden` and `immich`.

- [ ] **Step 5: Test Vaultwarden**

Navigate to `https://vaultwarden.tail654c62.ts.net` from any device on your tailnet. You should see the Vaultwarden login page with a valid HTTPS certificate (no browser warning).

- [ ] **Step 6: Test Immich**

Navigate to `https://immich.tail654c62.ts.net` from any device on your tailnet. You should see the Immich login page with a valid HTTPS certificate.
