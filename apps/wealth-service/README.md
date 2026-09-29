# wealth-service

Personal wealth tracking: a Spring Boot API backed by PostgreSQL, plus the
`wealth-frontend` UI. Both run in the `wealth-service` namespace.

```
wealth-service/       # Spring Boot API + PostgreSQL, exposed at https://wealth-service
  web-*.yaml          # wealth-frontend UI, exposed at https://wealth
```

The frontend talks to the API server-side through its `/api/wealth` proxy, so
the browser never calls the API directly and no CORS configuration is needed.

## Prerequisites

The database credentials must exist in Vault before the first apply, otherwise
the pods stay pending waiting on the Secret.

Create a KV v2 secret at path `wealth-service/database` with two keys:
`username` and `password`. These are used both for the PostgreSQL container
(`POSTGRES_USER` / `POSTGRES_PASSWORD`) and by the API
(`SPRING_DATASOURCE_USERNAME` / `SPRING_DATASOURCE_PASSWORD`).

`POSTGRES_DB` is fixed to `wealth` by `postgres.yaml`, and the API connects to
`jdbc:postgresql://wealth-service-postgres:5432/wealth`, so the database name
is taken care of. The `username` becomes the PostgreSQL superuser and must
already exist in the `wealth` database, which the container creates on first
boot.

GHCR pull credentials come from the shared `ghcr` Vault entry, as for every app.

## Apply

```bash
kubectl apply -k apps/wealth-service/
kubectl get pods -n wealth-service
```

## Images

Both images are built by GitHub Actions on push to `main` and the tag is
bumped automatically in `kustomization.yaml`:

- `ghcr.io/pestana1213/wealth-service` from `~/Desktop/Projects/wealth-service`
- `ghcr.io/pestana1213/wealth-frontend` from `~/Desktop/Projects/wealth-frontend`

## Metrics

The API exports `/actuator/prometheus` and is scraped by the `wealth-service`
ServiceMonitor every 30s. The frontend is not instrumented.
