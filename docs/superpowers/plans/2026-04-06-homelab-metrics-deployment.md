# Homelab Metrics Deployment Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Deploy the `homelab-metrics` application to Kubernetes so it collects host metrics and publishes them to the in-cluster Mosquitto MQTT broker.

**Architecture:** Raw Kubernetes manifests managed with Kustomize, following the same pattern as other apps in `apps/`. The existing `k8s/deployment.yaml` from the `homelab-metrics` repo is adapted: namespace added and `HOMELAB_MQTT_BROKER_URL` updated from a hardcoded IP/NodePort to the internal cluster DNS of the Mosquitto service.

**Tech Stack:** `ghcr.io/joaonogureira/homelab-metrics:latest`, Kubernetes manifests, Kustomize

---

## File Map

| Action | Path | Responsibility |
|--------|------|----------------|
| Create | `apps/homelab-metrics/namespace.yaml` | Namespace `homelab-metrics` |
| Create | `apps/homelab-metrics/deployment.yaml` | Deployment with host access and internal MQTT URL |
| Create | `apps/homelab-metrics/kustomization.yaml` | Kustomize resource list |

---

### Task 1: Namespace

**Files:**
- Create: `apps/homelab-metrics/namespace.yaml`

- [ ] **Step 1: Create namespace manifest**

```yaml
# apps/homelab-metrics/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: homelab-metrics
```

- [ ] **Step 2: Commit**

```bash
git add apps/homelab-metrics/namespace.yaml
git commit -m "feat(homelab-metrics): add namespace"
```

---

### Task 2: Deployment

**Files:**
- Create: `apps/homelab-metrics/deployment.yaml`

- [ ] **Step 1: Create deployment manifest**

```yaml
# apps/homelab-metrics/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: homelab-metrics
  namespace: homelab-metrics
  labels:
    app: homelab-metrics
spec:
  replicas: 1
  selector:
    matchLabels:
      app: homelab-metrics
  template:
    metadata:
      labels:
        app: homelab-metrics
    spec:
      hostPID: true
      hostNetwork: true
      containers:
        - name: homelab-metrics
          image: ghcr.io/joaonogureira/homelab-metrics:latest
          env:
            - name: HOMELAB_MQTT_BROKER_URL
              value: "tcp://mosquitto.mosquitto.svc.cluster.local:1883"
            - name: HOMELAB_METRICS_DISK_PATH
              value: "/"
            - name: HOMELAB_METRICS_PUBLISH_INTERVAL_MS
              value: "5000"
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
            limits:
              memory: "256Mi"
              cpu: "200m"
          volumeMounts:
            - name: proc
              mountPath: /proc
              readOnly: true
            - name: sys
              mountPath: /sys
              readOnly: true
      volumes:
        - name: proc
          hostPath:
            path: /proc
        - name: sys
          hostPath:
            path: /sys
```

- [ ] **Step 2: Commit**

```bash
git add apps/homelab-metrics/deployment.yaml
git commit -m "feat(homelab-metrics): add deployment"
```

---

### Task 3: Kustomization

**Files:**
- Create: `apps/homelab-metrics/kustomization.yaml`

- [ ] **Step 1: Create kustomization manifest**

```yaml
# apps/homelab-metrics/kustomization.yaml
resources:
  - namespace.yaml
  - deployment.yaml
```

- [ ] **Step 2: Verify kustomize build is valid**

```bash
kubectl kustomize apps/homelab-metrics/
```

Expected: both manifests printed with no errors.

- [ ] **Step 3: Commit**

```bash
git add apps/homelab-metrics/kustomization.yaml
git commit -m "feat(homelab-metrics): add kustomization"
```

---

### Task 4: Apply and verify

- [ ] **Step 1: Apply via kustomize**

```bash
kubectl apply -k apps/homelab-metrics/
kubectl get all -n homelab-metrics
```

Expected: namespace active, pod in Running state.

- [ ] **Step 2: Check logs for successful MQTT publishing**

```bash
kubectl logs -n homelab-metrics -l app=homelab-metrics --follow
```

Expected: logs showing metrics being published to `mosquitto.mosquitto.svc.cluster.local:1883` every 5 seconds, no connection errors.

- [ ] **Step 3: Final commit**

```bash
git add .
git commit -m "feat(homelab-metrics): complete deployment"
```
