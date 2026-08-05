# ESO Setup along with IRSA

1. Update trust relationship file with OIDC and account number and create role

```bash
aws iam create-role \
  --role-name ESO-Vault-Role \
  --assume-role-policy-document file://trust-policy.json
```

2. Create service account with role arn under annotation

```bash
kubectl apply -f service-account.yaml
```

3. Enable external secret operator application under kustomization
