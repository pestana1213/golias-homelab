# Mosquitto Deployment Design

**Date:** 2026-04-06

## Summary

Add a Mosquitto MQTT broker to the homelab Kubernetes cluster for internal service communication.

## Requirements

- Internal access only (no external exposure via Tailscale)
- Ephemeral (no persistent storage)
- Anonymous connections allowed (no authentication)

## Architecture

Raw Kubernetes manifests managed with Kustomize, following the same pattern as existing apps (e.g., vaultwarden).

## Files

All files under `apps/mosquitto/`:

| File | Purpose |
|------|---------|
| `namespace.yaml` | Creates the `mosquitto` namespace |
| `configmap.yaml` | Contains `mosquitto.conf` with anonymous access and listener on port 1883 |
| `deployment.yaml` | Single replica using `eclipse-mosquitto:2`, mounts ConfigMap at `/mosquitto/config/` |
| `service.yaml` | ClusterIP service on port 1883 (MQTT protocol) |
| `kustomization.yaml` | Lists all four resources above |

## Networking

- Service type: `ClusterIP`
- Port: `1883` (standard MQTT)
- Internal DNS: `mosquitto.mosquitto.svc.cluster.local:1883`

## What is NOT included

- No PVC (ephemeral)
- No Secret (anonymous auth)
- No Ingress (internal only)
- No helmfile entry (raw manifests only)
