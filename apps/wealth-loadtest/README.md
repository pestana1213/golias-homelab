# wealth-loadtest

k6 load test for `wealth-service`, isolated from the production database.

The `POST /transactions` write path is the hot path in this service, so that is
what the test drives. `setup()` onboards users, accounts and buckets over HTTP
and the measured scenarios then hammer the write path.

## Isolation

Load-test data never touches the production database.

- Runs in its own namespace, `wealth-loadtest`, with no Ingress, so it is not
  reachable from the tailnet.
- Writes to a separate database, `wealth_loadtest`, on the **same PostgreSQL
  instance** as production. The schema is created by Liquibase on boot, so no
  SQL setup is needed.
- The database lives on the same pod and the same volume as production data, but
  in a different database. Production rows are not visible to it and vice versa.
  If you want storage isolation as well, give this its own PostgreSQL and PVC.
- OpenTelemetry export is disabled (`OTEL_SDK_DISABLED`), so load-test traces do
  not reach Alloy and OpenObserve. Prometheus still scrapes
  `/actuator/prometheus`, but the ServiceMonitor is not deployed by default, so
  load-test series do not land in the shared Prometheus and OpenObserve.
- The database credentials come from the same Vault entry as production
  (`wealth-service/database`). Secrets are namespaced, so this namespace has its
  own `ExternalSecret` pointing at it.

The one shared resource is the PostgreSQL pod, which stays at its current
`500m` CPU limit. A heavy enough run can therefore affect production latency.
Watch the `wealth-service-postgres` pod if that matters, or give the load test
its own database instance.

## Prerequisites

The `wealth-loadtest-scripts` ConfigMap holds the k6 scripts and is **not** in
this directory, to avoid duplicating them from the `wealth-service` repository.
Generate it from `~/Desktop/Projects/wealth-service/load-tests` before applying:

```bash
cd ~/Desktop/Projects/wealth-service/load-tests
kubectl create configmap wealth-loadtest-scripts \
  --from-file=wealth-service.js=wealth-service.js \
  --from-file=config.js=lib/config.js \
  --from-file=api.js=lib/api.js \
  --from-file=workload.js=lib/workload.js \
  --from-file=uuid.js=lib/uuid.js \
  -n wealth-loadtest --dry-run=client -o yaml | kubectl apply -f -
```

Regenerate it whenever the scripts change. `k6-cronjob.yaml` remaps the
flattened ConfigMap keys back into `lib/` so the relative imports resolve.

## Deploy

Order matters here. The app runs Liquibase against `wealth_loadtest`, so the
create-database Job has to finish first, and the Job reads the credentials from
a Secret that `ExternalSecret` has to create first.

```bash
kubectl apply -k apps/wealth-loadtest/

# Credentials, synced from Vault.
kubectl wait --for=condition=Ready secret/wealth-loadtest-db \
  -n wealth-loadtest --timeout=2m

# Then the database itself.
kubectl wait --for=condition=complete job/wealth-loadtest-create-database \
  -n wealth-loadtest --timeout=2m

# Then the API, which migrates the schema on boot.
kubectl rollout status deployment/wealth-service-loadtest \
  -n wealth-loadtest --timeout=3m
```

The create-database Job is idempotent, so re-applying is harmless.

## Run

The k6 workload is a `CronJob` with `suspend: true`, so it runs only when you
trigger it. To run once:

```bash
kubectl create job --from=cronjob/wealth-loadtest-k6 k6-run \
  -n wealth-loadtest
kubectl logs -f job/k6-run -n wealth-loadtest
```

Re-running is the same two commands. `concurrencyPolicy: Forbid` means a second
trigger is ignored while a run is in flight, and `ttlSecondsAfterFinished` clears
the finished pod after 15 minutes.

```bash
kubectl get jobs -n wealth-loadtest
kubectl describe job k6-run -n wealth-loadtest   # on failure
```

k6 exits non-zero when a threshold is breached, so a failed run shows up as a
failed Job. The summary in the logs is the result; the `conflict_rate` and
`transaction_duration` values are the interesting ones.

## Tuning

Edit the `env` block in `k6-cronjob.yaml`, then re-apply the CronJob so new Jobs
pick the values up. The knobs are documented in `load-tests/README.md` in the
`wealth-service` repository. The defaults deliberately set `SHARED_USER=true`,
which points every VU at user #1's accounts to maximise `FOR UPDATE NOWAIT`
contention.

Watch these while the run progresses:

```bash
kubectl top pod -n wealth-loadtest
kubectl top pod -n wealth-service
```

The load-test API pod gets more CPU headroom than production (`2000m` vs
`500m`) on purpose: the run is meant to find the ceiling of the application and
database, not of the pod's CPU limit. Compare latency against production
numbers with that in mind.

## Metrics

`servicemonitor.yaml` is **not** included in the kustomization. The cluster
Prometheus sets `serviceMonitorSelectorNilUsesHelmValues: false` with no explicit
`serviceMonitorSelector`, which renders an empty selector and so picks up every
ServiceMonitor in the cluster no matter what labels it carries. Applying the file
would remote-write load-test series into the same OpenObserve and Grafana
dashboards as production.

To scrape the load-test API deliberately:

```bash
kubectl apply -f apps/wealth-loadtest/servicemonitor.yaml
# ... run the test, read the metrics ...
kubectl delete -f apps/wealth-loadtest/servicemonitor.yaml
```

Until then, read the server's own counters straight from the pod. These are
independent of k6 and confirm that a run actually reached the database:

```bash
kubectl exec -n wealth-loadtest deploy/wealth-service-loadtest -- \
  wget -qO- http://localhost:8080/actuator/metrics/wealth.transactions
kubectl exec -n wealth-loadtest deploy/wealth-service-loadtest -- \
  wget -qO- http://localhost:8080/actuator/metrics/wealth.http.requests
```

## Cleanup

```bash
kubectl delete -k apps/wealth-loadtest/

kubectl exec -n wealth-service deploy/wealth-service-postgres -- sh -c '
  psql -U "$POSTGRES_USER" -d postgres -v ON_ERROR_STOP=1 \
    -c "SELECT pg_terminate_backend(pid) FROM pg_stat_activity \
        WHERE datname = '\''wealth_loadtest'\'' AND pid <> pg_backend_pid()" \
    -c "DROP DATABASE IF EXISTS wealth_loadtest"
'
```

The first command removes the namespace and everything in it. The second is
needed because the load-test database lives on the production PostgreSQL pod
rather than in the namespace, so deleting the namespace does not remove it. It
reads the credentials straight from the pod's environment, so there is nothing
to substitute.

If the drop reports active connections, scale the load-test API down first so it
lets go of the pool:

```bash
kubectl scale deploy/wealth-service-loadtest -n wealth-loadtest --replicas=0
```