# golias-homelab

Kubernetes homelab configuration managed with Kustomize and Helmfile.

## Structure

```
helmfile.yaml       # Helm-managed cluster dependencies (operators, controllers)
apps/               # Application manifests (Kustomize)
  tailscale/        # Tailscale Kubernetes Operator
  kafka/            # Strimzi Kafka Operator
  immich/           # Photo management
  vaultwarden/      # Password manager
  adguard/          # DNS ad blocker
  monitoring/       # Grafana + Prometheus stack
  homePage/         # Homepage dashboard
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

## Ingress

Services are exposed via [Tailscale Ingress](https://tailscale.com/kb/1236/kubernetes-operator).
Each app with `ingressClassName: tailscale` gets its own Tailscale device with automatic HTTPS.
