# Netronome

Network speed testing and monitoring using the [official Docker image](https://github.com/autobrr/netronome#docker-installation), deployed with Kustomize.

## Deploy

Select your homelab Kubernetes context before running:

```bash
kubectl apply -k apps/netronome/
kubectl rollout status deployment/netronome -n netronome
kubectl get ingress -n netronome
```

Requires the Tailscale Kubernetes Operator and a default StorageClass supporting a 5Gi ReadWriteOnce volume.
Open the HTTPS hostname reported by the ingress on your tailnet and register your first account. The Tailscale device name is `netronome`; use its full tailnet hostname for HTTPS certificate matching.

To access the service without ingress:

```bash
kubectl port-forward -n netronome service/netronome 7575:80
```

Then open <http://localhost:7575>.

## Storage and networking

- Configuration and the default SQLite database persist under `/data` on `netronome-data`. Back up this volume; deleting the PVC may delete its data.
- One replica and the Recreate strategy prevent overlapping instances during upgrades.
- A root init container sets `/data` ownership to the image's bundled `netronome` user and group, including existing files. The application runs as that non-root user. Ownership is required because Netronome calls `chmod` on its database directory; supplemental group write access alone is insufficient. The storage backend must support `chown` (root-squashed NFS requires ownership provisioning on the storage server).
- `NET_RAW` enables raw sockets for network diagnostics. No host networking is enabled: tests measure connectivity from the pod, including the cluster network path. Host interface bandwidth monitoring requires a separately configured agent and vnstat.
- CPU is requested but not capped, to avoid throttling speed tests. Memory is capped at 512Mi.
- The image follows the upstream `latest` tag. To update, run `kubectl rollout restart deployment/netronome -n netronome`.

## Homepage

Apply the updated dashboard configuration and restart Homepage to copy it into the running pod:

```bash
kubectl apply -k apps/homePage/
kubectl rollout restart deployment/homepage -n homepage
```

The dashboard follows this repository's short Tailscale URL convention (`https://netronome`). If your browser requires the full tailnet hostname, update its `href` in `apps/homePage/homepage-stack.yaml` using the ingress address.
