# Tailscale Kubernetes Operator

Manages Tailscale ingress for homelab services. Each `Ingress` with `ingressClassName: tailscale` creates a Tailscale device automatically.

## Install / Upgrade

```bash
helmfile apply --set oauth.clientId=<CLIENT_ID> --set oauth.clientSecret=<CLIENT_SECRET>
```

## OAuth credentials

Generate at [login.tailscale.com/admin/settings/oauth](https://login.tailscale.com/admin/settings/oauth).
Required scopes: `devices:write`.
