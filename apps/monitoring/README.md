# Observability stack

The monitoring namespace provides Prometheus and Grafana for metrics, Loki for
logs, Tempo for traces, OpenObserve as an additional unified observability
backend, and Grafana Alloy as the node log collector and OTLP gateway.

## Instrumenting applications with OpenTelemetry

Applications with an OpenTelemetry SDK or auto-instrumentation can send OTLP to
Alloy through either of these in-cluster endpoints:

- gRPC: `http://alloy.monitoring.svc.cluster.local:4317`
- HTTP/protobuf: `http://alloy.monitoring.svc.cluster.local:4318`

Add these environment variables to an application Deployment (and install or
enable the OpenTelemetry SDK appropriate for its language):

```yaml
env:
  - name: OTEL_SERVICE_NAME
    value: my-service
  - name: OTEL_EXPORTER_OTLP_ENDPOINT
    value: http://alloy.monitoring.svc.cluster.local:4318
  - name: OTEL_EXPORTER_OTLP_PROTOCOL
    value: http/protobuf
  - name: OTEL_RESOURCE_ATTRIBUTES
    value: deployment.environment=homelab,service.namespace=my-namespace
```

Alloy sends traces to Tempo, metrics to Prometheus, and OTLP logs to Loki while
also duplicating all three signals to OpenObserve. Prometheus remote-writes its
scraped Kubernetes and application metrics to OpenObserve. Application
stdout/stderr is collected automatically and sent to both Loki and OpenObserve.
For log-to-trace links, include a 32-character trace ID in log messages using a
field such as `trace_id`.

OpenObserve is exposed through the Tailscale ingress named `openobserve`. Its
root credentials come from the Vault secret `openobserve`, which must contain:

- `root-email`: the root user's email address
- `root-password`: the root user's password
