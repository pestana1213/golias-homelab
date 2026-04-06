# Mosquitto Deployment Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Deploy an Eclipse Mosquitto MQTT broker on Kubernetes, accessible from ESP32 devices on the LAN via NodePort and from other cluster services via ClusterIP.

**Architecture:** Raw Kubernetes manifests managed with Kustomize, following the same pattern as other apps in this repo (e.g., `apps/vaultwarden/`). A ConfigMap provides `mosquitto.conf`, a Deployment runs the broker, and a NodePort Service exposes port 1883 both internally and on the cluster node's IP.

**Tech Stack:** `eclipse-mosquitto:2`, Kubernetes manifests, Kustomize

---

## File Map

| Action | Path | Responsibility |
|--------|------|----------------|
| Create | `apps/mosquitto/namespace.yaml` | Namespace `mosquitto` |
| Create | `apps/mosquitto/configmap.yaml` | `mosquitto.conf` — anonymous access, listener 1883 |
| Create | `apps/mosquitto/deployment.yaml` | 1-replica Deployment mounting the ConfigMap |
| Create | `apps/mosquitto/service.yaml` | NodePort Service on port 1883 |
| Create | `apps/mosquitto/kustomization.yaml` | Kustomize resource list |

---

### Task 1: Namespace

**Files:**
- Create: `apps/mosquitto/namespace.yaml`

- [ ] **Step 1: Create namespace manifest**

```yaml
# apps/mosquitto/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: mosquitto
```

- [ ] **Step 2: Apply and verify**

```bash
kubectl apply -f apps/mosquitto/namespace.yaml
kubectl get namespace mosquitto
```

Expected output:
```
namespace/mosquitto created
NAME        STATUS   AGE
mosquitto   Active   Xs
```

- [ ] **Step 3: Commit**

```bash
git add apps/mosquitto/namespace.yaml
git commit -m "feat(mosquitto): add namespace"
```

---

### Task 2: ConfigMap

**Files:**
- Create: `apps/mosquitto/configmap.yaml`

- [ ] **Step 1: Create ConfigMap manifest**

```yaml
# apps/mosquitto/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mosquitto-config
  namespace: mosquitto
data:
  mosquitto.conf: |
    listener 1883
    allow_anonymous true
```

- [ ] **Step 2: Apply and verify**

```bash
kubectl apply -f apps/mosquitto/configmap.yaml
kubectl get configmap mosquitto-config -n mosquitto
```

Expected output:
```
configmap/mosquitto-config created
NAME               DATA   AGE
mosquitto-config   1      Xs
```

- [ ] **Step 3: Commit**

```bash
git add apps/mosquitto/configmap.yaml
git commit -m "feat(mosquitto): add configmap with anonymous config"
```

---

### Task 3: Deployment

**Files:**
- Create: `apps/mosquitto/deployment.yaml`

- [ ] **Step 1: Create Deployment manifest**

```yaml
# apps/mosquitto/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mosquitto
  namespace: mosquitto
  labels:
    app: mosquitto
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mosquitto
  template:
    metadata:
      labels:
        app: mosquitto
    spec:
      containers:
        - name: mosquitto
          image: eclipse-mosquitto:2
          ports:
            - containerPort: 1883
          volumeMounts:
            - name: config
              mountPath: /mosquitto/config/mosquitto.conf
              subPath: mosquitto.conf
          resources:
            requests:
              memory: "32Mi"
              cpu: "50m"
            limits:
              memory: "64Mi"
              cpu: "100m"
      volumes:
        - name: config
          configMap:
            name: mosquitto-config
```

- [ ] **Step 2: Apply and verify pod is running**

```bash
kubectl apply -f apps/mosquitto/deployment.yaml
kubectl rollout status deployment/mosquitto -n mosquitto
kubectl get pods -n mosquitto
```

Expected output:
```
deployment.apps/mosquitto created
Waiting for deployment "mosquitto" rollout to finish: 0 of 1 updated replicas are available...
deployment "mosquitto" successfully rolled out
NAME                         READY   STATUS    RESTARTS   AGE
mosquitto-xxxxxxxxx-xxxxx    1/1     Running   0          Xs
```

- [ ] **Step 3: Verify config is loaded (check logs)**

```bash
kubectl logs -n mosquitto -l app=mosquitto
```

Expected: no errors, mosquitto starts listening on port 1883.

- [ ] **Step 4: Commit**

```bash
git add apps/mosquitto/deployment.yaml
git commit -m "feat(mosquitto): add deployment"
```

---

### Task 4: Service

**Files:**
- Create: `apps/mosquitto/service.yaml`

- [ ] **Step 1: Create Service manifest**

```yaml
# apps/mosquitto/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: mosquitto
  namespace: mosquitto
  labels:
    app: mosquitto
spec:
  type: NodePort
  ports:
    - port: 1883
      targetPort: 1883
      protocol: TCP
      name: mqtt
  selector:
    app: mosquitto
```

- [ ] **Step 2: Apply and note the assigned NodePort**

```bash
kubectl apply -f apps/mosquitto/service.yaml
kubectl get service mosquitto -n mosquitto
```

Expected output (NodePort will be auto-assigned between 30000–32767):
```
service/mosquitto created
NAME        TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
mosquitto   NodePort   10.x.x.x        <none>        1883:3XXXX/TCP   Xs
```

Note the assigned port (3XXXX) — this is the port your ESP32 will connect to on the node's IP.

- [ ] **Step 3: Get the node IP**

```bash
kubectl get nodes -o wide
```

Use the `INTERNAL-IP` column value as the broker address for your ESP32.

- [ ] **Step 4: Commit**

```bash
git add apps/mosquitto/service.yaml
git commit -m "feat(mosquitto): add nodeport service on port 1883"
```

---

### Task 5: Kustomization

**Files:**
- Create: `apps/mosquitto/kustomization.yaml`

- [ ] **Step 1: Create kustomization manifest**

```yaml
# apps/mosquitto/kustomization.yaml
resources:
  - namespace.yaml
  - configmap.yaml
  - deployment.yaml
  - service.yaml
```

- [ ] **Step 2: Verify kustomize build is valid**

```bash
kubectl kustomize apps/mosquitto/
```

Expected: all four manifests printed with no errors.

- [ ] **Step 3: Commit**

```bash
git add apps/mosquitto/kustomization.yaml
git commit -m "feat(mosquitto): add kustomization"
```

---

### Task 6: End-to-end verification

- [ ] **Step 1: Apply everything via kustomize**

```bash
kubectl apply -k apps/mosquitto/
kubectl get all -n mosquitto
```

Expected: namespace, pod (Running), service (NodePort) all present.

- [ ] **Step 2: Test MQTT connectivity from inside the cluster**

```bash
kubectl run mqtt-test --image=eclipse-mosquitto:2 --restart=Never -n mosquitto -- \
  mosquitto_pub -h mosquitto.mosquitto.svc.cluster.local -t test -m hello
kubectl logs mqtt-test -n mosquitto
kubectl delete pod mqtt-test -n mosquitto
```

Expected: no errors, message published successfully.

- [ ] **Step 3: Confirm NodePort and node IP for ESP32 configuration**

```bash
kubectl get svc mosquitto -n mosquitto -o jsonpath='{.spec.ports[0].nodePort}'
kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}'
```

Configure your ESP32 with:
- **Broker host:** `<node IP from above>`
- **Broker port:** `<nodePort from above>`

- [ ] **Step 4: Final commit**

```bash
git add .
git commit -m "feat(mosquitto): complete mosquitto deployment"
```
