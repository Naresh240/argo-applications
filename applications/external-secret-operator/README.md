# Integrating HashiCorp Vault with External Secrets Operator (ESO)

This guide demonstrates how to configure HashiCorp Vault with Kubernetes authentication so that External Secrets Operator (ESO) can securely retrieve an Artifactory token from Vault and create a Kubernetes Docker registry secret.

---

# Architecture

```
                    +---------------------------+
                    |      HashiCorp Vault      |
                    |---------------------------|
                    | Secret Engine (KV v2)     |
                    | Secret:                  |
                    | kv/artifactory-secret    |
                    |   └─ artifactory-token   |
                    +------------+-------------+
                                 |
                     Vault Policy (Read Access)
                                 |
                                 |
                      Kubernetes Auth Role
                 dynamic-secrets-pull-role
                                 |
                                 |
                    ServiceAccount Token
                                 |
                                 |
+--------------------------------------------------------------+
|                     Kubernetes Cluster                       |
|                                                              |
|  Namespace: argocd                                           |
|                                                              |
|  ServiceAccount                                              |
|  dynamic-secrets-pull-sa                                     |
|            |                                                 |
|            ▼                                                 |
|      SecretStore                                             |
|            |                                                 |
|            ▼                                                 |
|      ExternalSecret                                          |
|            |                                                 |
|            ▼                                                 |
| Kubernetes Secret (dockerconfigjson)                         |
| artifactory-docker-secret                                    |
+--------------------------------------------------------------+
```

---

# Prerequisites

- Kubernetes Cluster
- HashiCorp Vault
- Kubernetes Authentication enabled in Vault
- External Secrets Operator installed
- KV Version 2 Secret Engine mounted at `kv`

---

# Step 1: Create Vault Policy

Grant read access to the Artifactory secret.

```bash
vault policy write dynamic-secrets-pull-policy - <<EOF
path "kv/data/artifactory-secret" {
  capabilities = ["read"]
}
EOF
```

Verify the policy:

```bash
vault policy read dynamic-secrets-pull-policy
```

---

# Step 2: Create Kubernetes Service Account

```bash
kubectl create serviceaccount dynamic-secrets-pull-sa -n argocd
```

Verify:

```bash
kubectl get sa -n argocd
```

Expected output:

```
dynamic-secrets-pull-sa
```

---

# Step 3: Configure Vault Kubernetes Authentication

Configure the Kubernetes API endpoint.

```bash
vault write auth/kubernetes/config \
    kubernetes_host="https://<EKS_API_SERVER>"
```

Example:

```bash
vault write auth/kubernetes/config \
    kubernetes_host="https://XXXXXXXX.gr7.us-east-1.eks.amazonaws.com"
```

---

# Step 4: Create Vault Role

The role name **must match** the role configured in the SecretStore.

```bash
vault write auth/kubernetes/role/dynamic-secrets-pull-role \
    bound_service_account_names="dynamic-secrets-pull-sa" \
    bound_service_account_namespaces="argocd" \
    policies="dynamic-secrets-pull-policy" \
    audience="vault" \
    ttl="24h"
```

Verify:

```bash
vault read auth/kubernetes/role/dynamic-secrets-pull-role
```

---

# Step 5: Store the Artifactory Token

```bash
vault kv put kv/artifactory-secret \
    artifactory-token="<YOUR_ARTIFACTORY_TOKEN>"
```

Verify:

```bash
vault kv get kv/artifactory-secret
```

Expected output:

```
====== Data ======

artifactory-token    **************
```

---

# Step 6: Apply SecretStore

```bash
kubectl apply -f secretstore.yaml
```

SecretStore:

```yaml
apiVersion: external-secrets.io/v1
kind: SecretStore
metadata:
  name: vault-secret-store-1
  namespace: argocd
spec:
  provider:
    vault:
      server: https://vault.awsdevopstrainers.info
      path: kv
      version: v2
      auth:
        kubernetes:
          mountPath: kubernetes
          role: dynamic-secrets-pull-role
          serviceAccountRef:
            name: dynamic-secrets-pull-sa
```

---

# Step 7: Apply ExternalSecret

```bash
kubectl apply -f externalsecret.yaml
```

The ExternalSecret retrieves the Artifactory token from Vault and creates a Docker registry secret.

---

# Verification

## Check SecretStore

```bash
kubectl get secretstore -n argocd
```

Describe:

```bash
kubectl describe secretstore vault-secret-store-1 -n argocd
```

---

## Check ExternalSecret

```bash
kubectl get externalsecret -n argocd
```

Describe:

```bash
kubectl describe externalsecret artifactory-docker-secret -n argocd
```

---

## Check Generated Secret

```bash
kubectl get secret artifactory-docker-secret -n argocd
```

View the generated Docker configuration:

```bash
kubectl get secret artifactory-docker-secret \
    -n argocd \
    -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d
```

Expected output:

```json
{
  "auths": {
    "artifactory.awsdevopstrainers.info": {
      "username": "admin",
      "password": "<TOKEN>",
      "auth": "YWRtaW46PFRPS0VOPg=="
    }
  }
}
```

---

# Authentication Flow

```
External Secrets Operator
            │
            ▼
SecretStore
            │
            ▼
ServiceAccount
(dynamic-secrets-pull-sa)
            │
            ▼
Vault Kubernetes Authentication
            │
            ▼
Vault Role
(dynamic-secrets-pull-role)
            │
            ▼
Vault Policy
(dynamic-secrets-pull-policy)
            │
            ▼
KV Secret
(kv/artifactory-secret)
            │
            ▼
artifactory-token
            │
            ▼
Kubernetes Secret
artifactory-docker-secret
```

---

# Troubleshooting

### SecretStore is not Ready

Check:

```bash
kubectl describe secretstore vault-secret-store-1 -n argocd
```

Common causes:

- Incorrect Vault URL
- Incorrect role name
- Kubernetes authentication not configured
- Invalid ServiceAccount

---

### Permission Denied

Verify the Vault policy:

```bash
vault policy read dynamic-secrets-pull-policy
```

Ensure it grants access to:

```
kv/data/artifactory-secret
```

---

### Secret Not Found

Verify the secret exists:

```bash
vault kv get kv/artifactory-secret
```

---

### Authentication Failure

Verify the Vault role:

```bash
vault read auth/kubernetes/role/dynamic-secrets-pull-role
```

Confirm:

- Role name matches the SecretStore.
- ServiceAccount name is `dynamic-secrets-pull-sa`.
- Namespace is `argocd`.

---

# Summary

| Component | Value |
|-----------|-------|
| Vault Secret Engine | kv (KV v2) |
| Vault Secret | kv/artifactory-secret |
| Secret Property | artifactory-token |
| Vault Policy | dynamic-secrets-pull-policy |
| Vault Role | dynamic-secrets-pull-role |
| Kubernetes ServiceAccount | dynamic-secrets-pull-sa |
| Namespace | argocd |
| SecretStore | vault-secret-store-1 |
| ExternalSecret | artifactory-docker-secret |
| Generated Secret | artifactory-docker-secret |
