# Vaultwarden Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Deploy Vaultwarden password manager to the homelab Kubernetes cluster at `http://password.golias.home`.

**Architecture:** Multi-file Kubernetes manifests with kustomization, following the immich pattern. Single Vaultwarden container with SQLite stored on a 1Gi PVC. Admin token stored as a Kubernetes Secret.

**Tech Stack:** Kubernetes, Kustomize, Vaultwarden (`vaultwarden/server:latest`), Traefik ingress.

---

## File Map

| File | Action | Responsibility |
|------|--------|----------------|
| `apps/vaultwarden/namespace.yaml` | Create | `vaultwarden` namespace |
| `apps/vaultwarden/secret.yaml` | Create | Admin token secret |
| `apps/vaultwarden/deployment.yaml` | Create | PVC + Deployment (two documents) |
| `apps/vaultwarden/service.yaml` | Create | ClusterIP service on port 80 |
| `apps/vaultwarden/ingress.yaml` | Create | Ingress for `password.golias.home` |
| `apps/vaultwarden/kustomization.yaml` | Create | Kustomize resource list |

Reference: `apps/immich/` for the established pattern this follows.

---

### Task 1: Create namespace.yaml

**Files:**
- Create: `apps/vaultwarden/namespace.yaml`

- [ ] **Step 1: Create the file**

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: vaultwarden
```

- [ ] **Step 2: Validate**

```bash
kubectl apply --dry-run=client -f apps/vaultwarden/namespace.yaml
```

Expected: `namespace/vaultwarden configured (dry run)`

- [ ] **Step 3: Commit**

```bash
git add apps/vaultwarden/namespace.yaml
git commit -m "feat(vaultwarden): add namespace"
```

---

### Task 2: Create secret.yaml

**Files:**
- Create: `apps/vaultwarden/secret.yaml`

- [ ] **Step 1: Create the file**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: vaultwarden-secret
  namespace: vaultwarden
type: Opaque
stringData:
  admin-token: changeme
```

> **Note:** Replace `changeme` with a strong random token before deploying. Vaultwarden 1.28.0+ logs a deprecation warning when a raw (unhashed) token is used — this is cosmetic and does not affect functionality for homelab use.

- [ ] **Step 2: Validate**

```bash
kubectl apply --dry-run=client -f apps/vaultwarden/secret.yaml
```

Expected: `secret/vaultwarden-secret configured (dry run)`

- [ ] **Step 3: Commit**

```bash
git add apps/vaultwarden/secret.yaml
git commit -m "feat(vaultwarden): add admin token secret"
```

---

### Task 3: Create deployment.yaml

**Files:**
- Create: `apps/vaultwarden/deployment.yaml`

This file contains two documents separated by `---`: PVC first, Deployment second. This matches the immich pattern (`apps/immich/deployment.yaml`).

- [ ] **Step 1: Create the file**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: vaultwarden-data-pvc
  namespace: vaultwarden
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vaultwarden
  namespace: vaultwarden
  labels:
    app: vaultwarden
spec:
  replicas: 1
  selector:
    matchLabels:
      app: vaultwarden
  template:
    metadata:
      labels:
        app: vaultwarden
    spec:
      containers:
        - name: vaultwarden
          image: vaultwarden/server:latest
          ports:
            - containerPort: 80
          env:
            - name: ADMIN_TOKEN
              valueFrom:
                secretKeyRef:
                  name: vaultwarden-secret
                  key: admin-token
            - name: DOMAIN
              value: http://password.golias.home
            - name: SIGNUPS_ALLOWED
              value: "true"
          volumeMounts:
            - name: data
              mountPath: /data
          resources:
            requests:
              memory: "128Mi"
              cpu: "250m"
            limits:
              memory: "256Mi"
              cpu: "500m"
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: vaultwarden-data-pvc
```

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
git commit -m "feat(vaultwarden): add PVC and deployment"
```

---

### Task 4: Create service.yaml

**Files:**
- Create: `apps/vaultwarden/service.yaml`

- [ ] **Step 1: Create the file**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: vaultwarden
  namespace: vaultwarden
  labels:
    app: vaultwarden
spec:
  type: ClusterIP
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
      name: http
  selector:
    app: vaultwarden
```

- [ ] **Step 2: Validate**

```bash
kubectl apply --dry-run=client -f apps/vaultwarden/service.yaml
```

Expected: `service/vaultwarden configured (dry run)`

- [ ] **Step 3: Commit**

```bash
git add apps/vaultwarden/service.yaml
git commit -m "feat(vaultwarden): add service"
```

---

### Task 5: Create ingress.yaml

**Files:**
- Create: `apps/vaultwarden/ingress.yaml`

- [ ] **Step 1: Create the file**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: vaultwarden-ingress
  namespace: vaultwarden
spec:
  rules:
    - host: password.golias.home
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: vaultwarden
                port:
                  number: 80
```

- [ ] **Step 2: Validate**

```bash
kubectl apply --dry-run=client -f apps/vaultwarden/ingress.yaml
```

Expected: `ingress.networking.k8s.io/vaultwarden-ingress configured (dry run)`

- [ ] **Step 3: Commit**

```bash
git add apps/vaultwarden/ingress.yaml
git commit -m "feat(vaultwarden): add ingress for password.golias.home"
```

---

### Task 6: Create kustomization.yaml

**Files:**
- Create: `apps/vaultwarden/kustomization.yaml`

- [ ] **Step 1: Create the file**

```yaml
resources:
  - namespace.yaml
  - secret.yaml
  - deployment.yaml
  - service.yaml
  - ingress.yaml
```

- [ ] **Step 2: Validate kustomize build**

```bash
kubectl kustomize apps/vaultwarden/
```

Expected: All 6 resources printed (Namespace, Secret, PVC, Deployment, Service, Ingress) with no errors.

- [ ] **Step 3: Commit**

```bash
git add apps/vaultwarden/kustomization.yaml
git commit -m "feat(vaultwarden): add kustomization"
```

---

### Task 7: Deploy and verify

- [ ] **Step 1: Apply to cluster**

```bash
kubectl apply -k apps/vaultwarden/
```

Expected:
```
namespace/vaultwarden created
secret/vaultwarden-secret created
persistentvolumeclaim/vaultwarden-data-pvc created
deployment.apps/vaultwarden created
service/vaultwarden created
ingress.networking.k8s.io/vaultwarden-ingress created
```

- [ ] **Step 2: Wait for pod to be ready**

```bash
kubectl rollout status deployment/vaultwarden -n vaultwarden
```

Expected: `deployment "vaultwarden" successfully rolled out`

- [ ] **Step 3: Verify pod is running**

```bash
kubectl get pods -n vaultwarden
```

Expected: One pod with status `Running`.

- [ ] **Step 4: Open the UI**

Navigate to `http://password.golias.home` — Vaultwarden login page should appear.

- [ ] **Step 5: Create your account**

Register at `http://password.golias.home` (signups are open).

- [ ] **Step 6: Log in to admin panel**

Navigate to `http://password.golias.home/admin`, enter the token from `secret.yaml`.

- [ ] **Step 7: Disable signups**

Admin panel → General Settings → uncheck "Allow new signups" → Save.

> Until this step is done, any user on the local network can self-register.
