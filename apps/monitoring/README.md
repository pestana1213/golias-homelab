# Observability stack

The monitoring namespace provides Prometheus and Grafana for metrics, Loki for
logs, Tempo for traces, and Grafana Alloy as the node log collector and OTLP
gateway.

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

Alloy sends traces to Tempo, metrics to Prometheus, and OTLP logs to Loki.
Application stdout/stderr is also collected automatically and sent to Loki. For
log-to-trace links in Grafana, include a 32-character trace ID in log messages
using a field such as `trace_id`.
