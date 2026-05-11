# golias-homelab

Kubernetes homelab configuration managed with Kustomize and Helmfile.

## Structure

```
helmfile.yaml           # Helm-managed cluster dependencies (operators, controllers)
apps/
  tailscale/            # Tailscale Kubernetes Operator
  kafka/                # Strimzi Kafka Operator
  immich/               # Photo management
  vaultwarden/          # Password manager
  adguard/              # DNS ad blocker
  monitoring/           # Grafana + Prometheus stack
  homePage/             # Homepage dashboard
  vault/                # HashiCorp Vault — central secret store
  external-secrets/     # External Secrets Operator — syncs Vault secrets to Kubernetes
  ...
```

## Helm releases (helmfile.yaml)

Cluster-level dependencies are managed with [Helmfile](https://helmfile.readthedocs.io).

```bash
helmfile apply --set oauth.clientId=<CLIENT_ID> --set oauth.clientSecret=<CLIENT_SECRET>
```

See each app's `README.md` for required secrets and configuration.

## App manifests (apps/)

Individual apps are managed with [Kustomize](https://kustomize.io).

```bash
# Apply a single app
kubectl apply -k apps/<app-name>/

# Apply all apps
for dir in apps/*/; do kubectl apply -k "$dir"; done
```

## Mosquitto (MQTT)

Mosquitto is exposed via NodePort for ESP32 devices on the LAN. To find the assigned port and node IP:

```bash
# Get the NodePort
kubectl get svc mosquitto -n mosquitto -o jsonpath='{.spec.ports[0].nodePort}'

# Get the node IP
kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}'
```

Configure your ESP32 with:
- **Broker host:** `<node IP>`
- **Broker port:** `<NodePort>`

## Secret management

Secrets are managed with [HashiCorp Vault](apps/vault/README.md) and External Secrets Operator.

- Vault UI available at `https://vault` on the Tailscale network (login with root token)
- Vault **reseals on every pod restart** — unseal with:
  ```bash
  kubectl exec -n vault vault-helm-0 -- vault operator unseal <unseal-key>
  ```
- Store the unseal key and root token in Vaultwarden
- All app secrets are defined as `ExternalSecret` resources — ESO syncs them from Vault automatically

## Ingress

Services are exposed via [Tailscale Ingress](https://tailscale.com/kb/1236/kubernetes-operator).
Each app with `ingressClassName: tailscale` gets its own Tailscale device with automatic HTTPS.
