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

## Updating a secret

1. In Vault UI: **Secrets → homelab → \<path\> → Create new version**
2. Update the value and save
3. ESO syncs automatically within the `refreshInterval` (default 1h), or force it immediately:
   ```bash
   kubectl annotate externalsecret <name> -n <namespace> force-sync=$(date +%s) --overwrite
   ```

If the secret is a database password, also update it in the database and restart the app:
```bash
kubectl exec -n <namespace> <postgres-pod> -- psql -U postgres -c \
  "ALTER USER postgres WITH PASSWORD 'newpassword';"
kubectl rollout restart deployment/<app> -n <namespace>
```

## Monitoring with k9s

- `:externalsecrets` — check sync status across all namespaces, `SecretSynced` = healthy
- `:clustersecretstores` — confirm `vault` shows `Valid` and `Ready: True`

## Vault sealed after restart

Every time `vault-helm-0` restarts (node reboot, crash, update), Vault becomes **sealed** and all ExternalSecrets will show `SecretSyncedError` in k9s. Unseal with:
```bash
kubectl exec -n vault vault-helm-0 -- vault operator unseal <unseal-key>
```

Store the unseal key and root token in Vaultwarden.

## Vault access

Available via Tailscale at `https://vault`. Login with the root token.
