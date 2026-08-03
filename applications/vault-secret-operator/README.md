# HashiCorp Vault Secrets Operator (VSO) Integration

This document explains how to configure HashiCorp Vault Secrets Operator (VSO) to securely retrieve secrets from HashiCorp Vault using Kubernetes Authentication and sync them into Kubernetes Secrets.

---

# Architecture

```text
                    HashiCorp Vault
                  +------------------+
                  | KV Secret Engine |
                  |------------------|
                  | artifactory-secret
                  |  └── artifactory-token
                  +---------+--------+
                            |
                     Vault Policy
                            |
                     Kubernetes Auth
                            |
              Role: dynamic-secrets-pull-role
                            |
                            |
                 ServiceAccount Token
                            |
+--------------------------------------------------------------+
|                      Kubernetes Cluster                      |
|                                                              |
|  Namespace: argocd                                           |
|                                                              |
|  ServiceAccount                                              |
|  dynamic-secrets-pull-sa                                     |
|            │                                                 |
|            ▼                                                 |
|      VaultConnection                                         |
|            │                                                 |
|            ▼                                                 |
|        VaultAuth                                             |
|            │                                                 |
|            ▼                                                 |
|     VaultStaticSecret                                        |
|            │                                                 |
|            ▼                                                 |
| Kubernetes Secret                                            |
| artifactory-secret                                           |
+--------------------------------------------------------------+
```

---

# Prerequisites

- Kubernetes cluster
- HashiCorp Vault
- Vault Secrets Operator installed
- Kubernetes authentication enabled in Vault
- KV Version 2 secret engine mounted at `kv`

---

# Step 1: Create Vault Policy

Create a policy that allows the operator to read the Artifactory secret.

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

Create the service account referenced by `VaultAuth`.

```bash
kubectl create serviceaccount dynamic-secrets-pull-sa -n argocd
```

Verify:

```bash
kubectl get serviceaccount -n argocd
```

---

# Step 3: Configure Vault Kubernetes Authentication

Configure the Kubernetes authentication backend.

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

The role name must match the role configured in the `VaultAuth` resource.

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

# Step 5: Store Secret in Vault

Store the Artifactory token in Vault.

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

artifactory-token    *************
```

---

# Step 6: Deploy VaultConnection

```yaml
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultConnection
metadata:
  name: vault-connection
spec:
  address: https://vault.awsdevopstrainers.info
```

Apply:

```bash
kubectl apply -f vault-connection.yaml
```

---

# Step 7: Deploy VaultAuth

```yaml
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultAuth
metadata:
  name: vault-auth
spec:
  vaultConnectionRef: vault-connection
  method: kubernetes
  mount: kubernetes
  kubernetes:
    role: dynamic-secrets-pull-role
    serviceAccount: dynamic-secrets-pull-sa
```

Apply:

```bash
kubectl apply -f vault-auth.yaml
```

---

# Step 8: Deploy VaultStaticSecret

```yaml
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultStaticSecret
metadata:
  name: vault-artifactory-token
spec:
  vaultAuthRef: vault-auth
  mount: kv
  type: kv-v2
  path: artifactory-secret
  destination:
    create: true
    name: artifactory-secret
    type: Opaque
  refreshAfter: 5m
```

Apply:

```bash
kubectl apply -f vault-static-secret.yaml
```

---

# Verification

## Verify Vault Resources

```bash
kubectl get vaultconnection
kubectl get vaultauth
kubectl get vaultstaticsecret
```

---

## Describe VaultStaticSecret

```bash
kubectl describe vaultstaticsecret vault-artifactory-token
```

---

## Verify Kubernetes Secret

```bash
kubectl get secret artifactory-secret
```

View the secret:

```bash
kubectl get secret artifactory-secret -o yaml
```

Decode the token:

```bash
kubectl get secret artifactory-secret \
-o jsonpath='{.data.artifactory-token}' | base64 -d
```

---

# Synchronization Flow

```text
HashiCorp Vault
       │
       ▼
KV Secret Engine (kv)
       │
       ▼
artifactory-secret
       │
       ▼
Vault Policy
(dynamic-secrets-pull-policy)
       │
       ▼
Kubernetes Auth Role
(dynamic-secrets-pull-role)
       │
       ▼
ServiceAccount
(dynamic-secrets-pull-sa)
       │
       ▼
VaultConnection
       │
       ▼
VaultAuth
       │
       ▼
VaultStaticSecret
       │
       ▼
Kubernetes Secret
(artifactory-secret)
```

---

# Troubleshooting

## VaultConnection Not Ready

```bash
kubectl describe vaultconnection vault-connection
```

Verify:

- Vault URL is reachable
- TLS certificates are valid

---

## VaultAuth Authentication Failed

```bash
kubectl describe vaultauth vault-auth
```

Verify:

- Kubernetes auth method is enabled in Vault.
- Role name matches `dynamic-secrets-pull-role`.
- ServiceAccount is `dynamic-secrets-pull-sa`.
- Namespace matches the role configuration.

---

## Permission Denied

Verify the Vault policy:

```bash
vault policy read dynamic-secrets-pull-policy
```

The policy should include:

```hcl
path "kv/data/artifactory-secret" {
  capabilities = ["read"]
}
```

---

## Secret Not Created

Check the VaultStaticSecret status:

```bash
kubectl describe vaultstaticsecret vault-artifactory-token
```

Verify the secret exists in Vault:

```bash
vault kv get kv/artifactory-secret
```

---

# Summary

| Component | Value |
|-----------|-------|
| Vault Server | https://vault.awsdevopstrainers.info |
| Authentication Method | Kubernetes |
| Auth Mount | kubernetes |
| Vault Policy | dynamic-secrets-pull-policy |
| Vault Role | dynamic-secrets-pull-role |
| Kubernetes ServiceAccount | dynamic-secrets-pull-sa |
| Secret Engine | kv (KV v2) |
| Secret Path | artifactory-secret |
| Secret Property | artifactory-token |
| Kubernetes Secret | artifactory-secret |
| Refresh Interval | 5 minutes |

---

# Notes

- The Vault role (`dynamic-secrets-pull-role`) must exactly match the role specified in the `VaultAuth` resource.
- The Kubernetes ServiceAccount (`dynamic-secrets-pull-sa`) must exist in the same namespace where the `VaultAuth` and `VaultStaticSecret` resources are deployed.
- The Vault policy must grant read access to the KV v2 data path (`kv/data/artifactory-secret`).
- `refreshAfter: 5m` ensures the Kubernetes Secret is automatically synchronized with Vault every five minutes.