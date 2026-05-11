# Vault + External Secrets Operator

Vault is the central secret store for the homelab. External Secrets Operator (ESO) bridges Vault and Kubernetes — it reads secrets from Vault and creates native Kubernetes Secrets automatically.

## How it works

- **Vault path** (`remoteRef.key`) — where ESO reads the data in Vault (e.g. `homelab/immich`)
- **Kubernetes Secret name** (`spec.target.name`) — the name of the Secret ESO creates in the cluster

These are independent. Your pods reference the Kubernetes Secret name, not the Vault path.

```
Vault (homelab/immich)
        ↓  ESO reads
ExternalSecret (immich namespace)
        ↓  ESO creates
Kubernetes Secret "immich-secret"
        ↓  pod mounts
Immich deployment
```

## Adding a new secret

1. In Vault UI: **Secrets → homelab → Create secret**
   - Set the path (e.g. `my-app`)
   - Add key-value pairs

2. Create an `ExternalSecret` resource in the app's namespace:
   ```yaml
   apiVersion: external-secrets.io/v1beta1
   kind: ExternalSecret
   metadata:
     name: my-app-secret
     namespace: my-app
   spec:
     refreshInterval: 1h
     secretStoreRef:
       name: vault
       kind: ClusterSecretStore
     target:
       name: my-app-secret
     data:
       - secretKey: my-key
         remoteRef:
           key: my-app
           property: my-key
   ```

3. Remove the old `secret.yaml` from the kustomization and Git.

## Initial setup (one-time)

After Vault restarts it will be sealed. Unseal with:
```bash
kubectl exec -n vault vault-helm-0 -- vault operator unseal <unseal-key>
```

Store the unseal key and root token in Vaultwarden.

## Vault access

Available via Tailscale at `https://vault`. Login with the root token.
