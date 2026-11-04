# zitadel

Identity provider for every app in this cluster: one account per person, reused
across apps instead of a separate login per project. Runs in the `zitadel`
namespace with its own PostgreSQL, exposed at `https://zitadel` on the Tailscale
network.

Deployed from the official Helm chart through ArgoCD (`zitadel-stack.yaml`),
because the chart owns the init and setup jobs and the separate Login V2
workload. Everything else here follows the repo's own conventions: PostgreSQL as
a plain Deployment, secrets from Vault through External Secrets.

## Prerequisites

Three Vault entries must exist before the first apply, otherwise the pods wait
on their Secrets.

| Vault path (KV v2) | Keys | Notes |
| --- | --- | --- |
| `zitadel/database` | `username`, `password` | PostgreSQL superuser. The DSN is templated from these by `external-secret.yaml`. |
| `zitadel/masterkey` | `masterkey` | Exactly 32 characters. |
| `zitadel/login` | `sessionCookieSecret`, `tlsCrt`, `tlsKey` | Cookie signing secret, and the keypair the Login UI authenticates to ZITADEL with. |

Generate them:

```bash
tr -dc A-Za-z0-9 </dev/urandom | head -c 32   # masterkey
openssl rand -base64 32                        # sessionCookieSecret

# tlsCrt / tlsKey: the chart would generate these, but ArgoCD re-renders the
# chart on every sync and the generated keypair differs each time, which would
# rotate the login service key continuously. So it is pinned in Vault. Keep the
# subject and SAN, which is what the chart's own certificate uses.
openssl genrsa -traditional -out tls.key 2048
openssl req -new -x509 -key tls.key -out tls.crt -days 3650 \
  -subj "/CN=login-service" -addext "subjectAltName=DNS:login-service"
```

Store the PEM contents of `tls.crt` and `tls.key` as `tlsCrt` and `tlsKey`.

**The masterkey cannot be rotated.** It encrypts what ZITADEL stores, and losing
it means losing access to that data. Keep a copy in Vaultwarden next to the
Vault root token, and make sure the `zitadel` PostgreSQL volume is covered by
whatever backup the cluster grows.

## Set this before the first apply

`zitadel.configmapConfig.ExternalDomain` in `zitadel-stack.yaml` is a
placeholder (`zitadel.TAILNET.ts.net`). It must be the **real MagicDNS name** of
the Tailscale device, because ZITADEL generates its OIDC endpoints, redirect
URLs and cookie domain from it. The short `zitadel` host in the ingress is what
names the device; it is not what the browser sends.

```bash
kubectl get ingress -n zitadel      # once applied, to read the assigned name
```

## First login

The chart creates one admin from `FirstInstance.Org.Human` with username
`admin` and `PasswordChangeRequired: true`. The chart generates the initial
password; read it from the setup job's output or set one explicitly in
`valuesObject` before the first apply.

Registration is closed on purpose: accounts are created from the console by
hand. Opening self-registration later is a login-policy toggle in the console,
not a change here.

## Two things to verify on the first apply

Both come from this cluster using Tailscale ingress rather than the Traefik or
NGINX setups the chart documents:

1. **HTTP/2 to the pods.** ZITADEL serves its REST and gRPC APIs over h2c on one
   port. The chart sets `service.appProtocol: kubernetes.io/h2c` and the Traefik
   `serversscheme` annotation, which the Tailscale proxy ignores. The OIDC
   endpoints are plain HTTP and will work regardless; the admin console uses
   gRPC-web, so check it loads before relying on it.
2. **The two ingresses share one host.** `/ui/v2/login` must reach the login
   service and `/` the ZITADEL service. If the Tailscale operator creates two
   devices instead of one, merge them into a single Ingress with both paths.
