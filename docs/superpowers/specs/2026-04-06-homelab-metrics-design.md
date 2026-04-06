# Homelab Metrics Deployment Design

**Date:** 2026-04-06

## Summary

Add the `homelab-metrics` application to the golias-homelab Kubernetes manifests. It collects system metrics (CPU, memory, disk, network) from the host and publishes them to the Mosquitto MQTT broker.

## Requirements

- Deploy `ghcr.io/joaonogureira/homelab-metrics:latest` in the cluster
- Connect to the Mosquitto broker via internal cluster DNS
- Access host `/proc` and `/sys` for metrics collection
- No external access needed (publisher only)

## Architecture

Raw Kubernetes manifests managed with Kustomize, following the same pattern as other apps. No service or ingress — the app only publishes outbound to MQTT.

The existing `k8s/deployment.yaml` in the `homelab-metrics` repo is adapted: namespace added, and `HOMELAB_MQTT_BROKER_URL` updated from the hardcoded `tcp://192.168.1.225:31719` to the internal cluster DNS `tcp://mosquitto.mosquitto.svc.cluster.local:1883`.

## Files

All files under `apps/homelab-metrics/`:

| File | Purpose |
|------|---------|
| `namespace.yaml` | Creates the `homelab-metrics` namespace |
| `deployment.yaml` | Deployment with `hostPID`, `hostNetwork`, host volume mounts, and internal MQTT URL |
| `kustomization.yaml` | Lists both resources |

## Configuration

| Env var | Value |
|---------|-------|
| `HOMELAB_MQTT_BROKER_URL` | `tcp://mosquitto.mosquitto.svc.cluster.local:1883` |
| `HOMELAB_METRICS_DISK_PATH` | `/` |
| `HOMELAB_METRICS_PUBLISH_INTERVAL_MS` | `5000` |

## What is NOT included

- No Service (outbound publisher only)
- No Ingress
- No Secret
- No helmfile entry
