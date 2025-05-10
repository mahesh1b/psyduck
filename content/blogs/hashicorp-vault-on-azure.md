---
date: '2025-05-08T08:59:46+05:30'
draft: false
title: 'Hashicorp Va""ult on Azure AKS'
tags: ["vault", "Azure", "terraform", "AKS"]
cover: 
    image: images/hashicorp-vault-on-azure.jpg
    responsiveImages: true
    linkFullImages: true
---


In today's cloud-native ecosystem, securely managing secrets across dynamic workloads is essential. This guide walks through deploying and integrating HashiCorp Vault with Azure Kubernetes Service (AKS), providing a robust solution for secure secret storage, dynamic secret generation, and granular access control within a scalable, cloud-optimized environment.

### What is Vault?
HashiCorp Vault is a powerful secrets management solution that:
- Securely stores and tightly controls access to tokens, passwords, certificates, and encryption keys
- Provides both UI and CLI interfaces along with HTTP API
- Offers capabilities for:
    - Secret management
    - Database credentials management
    - Data encryption
    - Identity-based access

### Prerequisites
Before beginning, ensure you have:
1. A Microsoft Azure account with an AKS cluster deployed
2. Terraform installed and configured
3. Azure CLI installed
4. kubectl configured to access your AKS cluster

### Implementation Overview

1.  Azure Key Vault for Auto-Unseal
Vault servers start in a sealed state and require unsealing to decrypt data. We'll use Azure Key Vault for automatic unsealing:
```tf
resource "azurerm_key_vault" "key_vault" {
  name                = "foo-key-vault"
  location            = local.region
  resource_group_name = local.vault_rg
  tenant_id           = local.tenant_id
  sku_name            = "standard"

  # using Azure role-based access control for managing access to Azure key vault
  enable_rbac_authorization = true
}

resource "azurerm_key_vault_key" "vault_key" {
  name         = "foo-vault-unseal-key"
  key_vault_id = azurerm_key_vault.key_vault.id
  key_type     = "RSA"
  key_size     = 2048

  key_opts = [
    "decrypt",
    "encrypt",
    "sign",
    "unwrapKey",
    "verify",
    "wrapKey",
  ]
}
```

2. Azure Managed Identity for Access Control
We'll use Azure Managed Identity to securely grant Vault pods access to the Key Vault:
```tf
resource "azurerm_user_assigned_identity" "vault" {
  name                = "vault"
  resource_group_name = local.vault_rg
  location            = local.region
}

resource "azurerm_role_assignment" "vault_policy" {
  scope                = azurerm_key_vault.key_vault.id
  role_definition_name = "Key Vault Crypto User"
  principal_id         = azurerm_user_assigned_identity.vault.principal_id
}

resource "azurerm_federated_identity_credential" "vault_access" {
  name                = "vault-access"
  resource_group_name = local.vault_rg
  parent_id           = azurerm_user_assigned_identity.vault.id
  issuer              = data.azurerm_kubernetes_cluster.this.oidc_issuer_url
  subject             = "system:serviceaccount:vault:vault"
  audience            = ["api://AzureADTokenExchange"]
}
```

3. Deploying Vault with Helm
We'll use the official Vault Helm chart with custom values:
```tf
resource "helm_release" "vault" {
  name = "vault"

  namespace        = "vault"
  repository       = "https://helm.releases.hashicorp.com"
  chart            = "vault"
  version          = "0.29.1"
  create_namespace = true

  values = [templatefile("${path.module}/values.yaml", {
    client_id      = azurerm_user_assigned_identity.vault.client_id
    tenant_id      = local.tenant_id
    sub_id         = local.subscription_id
    vault_name     = azurerm_key_vault.key_vault.name
    vault_key      = azurerm_key_vault_key.vault_key.name
  })]
}
```
4. Helm Values Configuration
The values.yaml file configures Vault with:
    - Raft storage backend for HA
    - Azure Key Vault auto-unseal
    - Workload identity integration
```yaml
global:
  enabled: true

injector:
  enabled: false

server:
  enabled: false

  dataStorage:
    size: 2Gi

  extraLabels:
    azure.workload.identity/use: "true"

  ha:
    enabled: true
    replicas: 2
    raft:
      enabled: true
      config: |
        ui = true

        cluster_name = "vault-integrated-storage"
        storage "raft" {
            path    = "/vault/data/"
        }

        listener "tcp" {
            address = "[::]:8200"
            cluster_address = "[::]:8201"
            tls_disable = "true"
        }

        service_registration "kubernetes" {}

        seal "azurekeyvault" {
          tenant_id       = "${tenant_id}"
          vault_name      = "${vault_name}"
          key_name        = "${vault_key}"
          subscription_id = "${sub_id}"
        }
        
  serviceAccount:
    createSecret: true
    annotations:
      "azure.workload.identity/client-id": "${client_id}"
    extraLabels:
      "azure.workload.identity/use": "true"

ui:
  enabled: true
```

### Initializing and Configuring Vault
After deployment:
1. Check pod status:
```bash
kubectl get pods -n vault
```

2. Initialize Vault (on vault-0):
```bash
kubectl exec -it vault-0 -n vault -- vault operator init
```
This provides the root token and recovery keys - store these securely.

3. Join additional nodes to the Raft cluster (on vault-1):
```bash
kubectl exec -it vault-1 -n vault -- vault operator raft join http://vault-0.vault-internal:8200
```

### Conclusion
This deployment provides a highly available, secure Vault installation on AKS with:
- Automatic unsealing via Azure Key Vault
- Secure workload identity integration
- Raft-based high availability storage

The solution balances security with operational simplicity, making it suitable for production environments while maintaining the flexibility needed for modern cloud-native applications.