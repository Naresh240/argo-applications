# Dynamic JFrog Token Architecture using Vault Secrets Operator

## Overview

This architecture ensures that Kubernetes applications receive **short-lived Artifactory access tokens** while the **long-lived administrator/service token remains securely stored in Vault**.

```mermaid
┌──────────────────────────── JFrog Platform ─────────────────────────────┐
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                  JFrog Artifactory / Access Service              │   │
│  │                                                                  │   │
│  │  Long-lived Service/Admin Token                                  │   │
│  │                                                                  │   │
│  │  POST /access/api/v1/tokens                                      │   │
│  │      │                                                           │   │
│  │      ▼                                                           │   │
│  │  Generates Short-lived Token (TTL = 1 Hour)                      │   │
│  └──────────────────────────────────────────────────────────────────┘   │
└───────────────────────────────▲──────────────────────────────────────────┘
                                │ HTTPS
                                │
                                │
┌────────────────────────── HashiCorp Vault ───────────────────────────────┐
│                                                                          │
│  Kubernetes Auth                                                         │
│                                                                          │
│        ▲                                                                 │
│        │ Login using ServiceAccount                                      │
│        │                                                                 │
│  ┌──────────────────────────────────────────────────────────────┐        │
│  │           Artifactory Secret Engine / Plugin                 │        │
│  │                                                              │        │
│  │  Stores Admin Token                                          │        │
│  │                                                              │        │
│  │  Calls JFrog Access Token API                                │────────┘
│  │                                                              │
│  │  Returns                                                     │
│  │      access_token                                            │
│  │      lease_duration                                          │
│  │      renewable                                               │
│  └──────────────────────────────────────────────────────────────┘
│
└──────────────────────────────▲───────────────────────────────────────────┘
                               │
                               │ Vault API
                               │
┌──────────────────────────── Kubernetes ──────────────────────────────────┐
│                                                                          │
│  Vault Secrets Operator                                                  │
│                                                                          │
│  VaultDynamicSecret                                                      │
│         │                                                                │
│         ▼                                                                │
│  Creates Kubernetes Secret                                               │
│         │                                                                │
│         ▼                                                                │
│  Application Pod                                                         │
│         │                                                                │
│         ▼                                                                │
│  Uses Artifactory Token                                                  │
└──────────────────────────────────────────────────────────────────────────┘
```

## Authentication Flow

1. Vault Secrets Operator authenticates to Vault using the Kubernetes ServiceAccount.
2. Vault validates the ServiceAccount JWT using Kubernetes Authentication.
3. Vault returns a Vault Token.

## Secret Generation Flow

1. `VaultDynamicSecret` requests a secret from Vault.
2. Vault invokes the Artifactory Secrets Engine (Plugin).
3. The plugin retrieves the stored long-lived Artifactory admin/service token.
4. The plugin calls the JFrog Access Token API.
5. JFrog generates a short-lived access token (TTL = 1 hour).
6. Vault returns the token as a leased secret.
7. Vault Secrets Operator creates or updates the Kubernetes Secret.
8. The application consumes the Kubernetes Secret.

## Token Renewal

```text
Application
      │
      ▼
Kubernetes Secret
      ▲
      │
Vault Secrets Operator
      │
      ▼
Vault Dynamic Secret
      │
      ▼
Vault Plugin
      │
      ▼
JFrog Access Token API
      │
      ▼
Generate New Token
      │
      ▼
Update Kubernetes Secret
```

When the lease reaches the configured threshold (for example, `renewalPercent: 67`), Vault Secrets Operator requests a renewal. If the token cannot be renewed, the plugin generates a new access token from JFrog and updates the Kubernetes Secret automatically.

## Security Model

| Component | Responsibility |
|-----------|----------------|
| JFrog Artifactory | Generates short-lived access tokens |
| Vault Plugin | Stores the long-lived admin token and requests new access tokens |
| Vault | Authenticates Kubernetes and manages leases |
| Vault Secrets Operator | Synchronizes dynamic secrets into Kubernetes |
| Kubernetes | Consumes only short-lived access tokens |

## Benefits

- Long-lived administrator token never leaves Vault.
- Kubernetes workloads receive only short-lived access tokens.
- Automatic token rotation.
- Automatic Kubernetes Secret updates.
- Centralized auditing in Vault.