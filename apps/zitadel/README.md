# zitadel

Identity provider for every app in this cluster: one account per person, reused
across apps instead of a separate login per project. Runs in the `zitadel`
namespace with its own PostgreSQL, exposed at `https://zitadel` on the Tailscale
network.

Deployed from the official Helm chart through ArgoCD (`zitadel-stack.yaml`, the
Application `zitadel-helm` — named apart from the folder so it does not collide
with the per-directory Application that applies everything else here),
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
| `zitadel/admin` | `password` | The first admin's password. Reaches ZITADEL as a config fragment, so it is never in Git. |

The admin password must satisfy ZITADEL's default complexity policy: an
uppercase letter, a lowercase letter, a number and a symbol. A password that
does not fails the `03_default_instance` migration with
`Errors.User.PasswordComplexityPolicy.*`, and the setup job exits 1 before any
of the chart's own resources are created, which shows up in ArgoCD as every
resource `Missing`. A plain hex or base64 string is not enough on its own:

```bash
echo "$(openssl rand -base64 12)Aa1!"
```

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
# The suffix is the same one the other apps already use
tailscale status --json | jq -r .MagicDNSSuffix
kubectl get ingress -n zitadel      # once applied, to read the assigned name
```

`FirstInstance.Org.Human.Email.Address` is the other placeholder. Both are in
`zitadel-stack.yaml`.

## First login

The setup job creates one human admin from `FirstInstance.Org.Human`: username
`admin`, password from `zitadel/admin` in Vault, and `PasswordChangeRequired:
true`, so the first login asks for a new one.

`Email.Address` in `zitadel-stack.yaml` is a placeholder (`admin@CHANGEME`) and
must be a mailbox you read. It is marked `Verified: true` on purpose: no SMTP is
configured, so an unverified address would leave the account unable to finish
logging in.

The chart also creates a machine user `iam-admin` with `IAM_OWNER` and puts its
JWT key in a Secret named after it — that is the chart's own default, and it is
what you would use to drive the API from scripts.

Registration is closed on purpose: accounts are created from the console by
hand. Opening self-registration later is a login-policy toggle in the console,
not a change here.

## Ingress

Hand-written in `ingress.yaml`, with the chart's own ingresses disabled
(`ingress.enabled: false`, `login.ingress.enabled: false`).

The chart offers one Ingress per workload, but the Tailscale operator creates a
**device per Ingress**: two of them registered as `zitadel` and `zitadel-1`, so
`/ui/v2/login` ended up on a different hostname from the one `ExternalDomain`
names, and the login redirect left the host. One Ingress with both paths keeps
it to a single device.

Still worth checking: ZITADEL serves its REST and gRPC APIs over h2c on one
port. The chart sets `service.appProtocol: kubernetes.io/h2c` and a Traefik
`serversscheme` annotation, neither of which the Tailscale proxy reads. The OIDC
endpoints are plain HTTP and work regardless; the admin console uses gRPC-web,
so confirm it loads before relying on it.
