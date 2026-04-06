# Mosquitto Deployment Design

**Date:** 2026-04-06

## Summary

Add a Mosquitto MQTT broker to the homelab Kubernetes cluster for internal service communication.

## Requirements

- Accessible from the local network (LAN) for ESP32 devices on the same network as the cluster
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

- Service type: `NodePort`
- Port: `1883` (standard MQTT)
- NodePort: auto-assigned by Kubernetes (range 30000–32767)
- Internal DNS: `mosquitto.mosquitto.svc.cluster.local:1883`
- LAN access: `<node-IP>:<assigned-nodePort>` — ESP32 devices connect via this address

## What is NOT included

- No PVC (ephemeral)
- No Secret (anonymous auth)
- No Tailscale Ingress
- No helmfile entry (raw manifests only)
