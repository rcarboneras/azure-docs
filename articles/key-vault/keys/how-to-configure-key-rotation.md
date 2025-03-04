---
title: Configure cryptographic key auto-rotation in Azure Key Vault
description: Use this guide to learn how to configure automated the rotation of a key in Azure Key Vault
services: key-vault
author: msmbaldwin
manager: rkarlin
tags: 'rotation'
ms.service: key-vault
ms.subservice: keys
ms.topic: how-to
ms.date: 11/24/2021
ms.author: mbaldwin
---
# Configure cryptographic key auto-rotation in Azure Key Vault

## Overview
Automated cryptographic key rotation in [Key Vault](../general/overview.md) allows users to configure Key Vault to automatically generate a new key version at a specified frequency. To configure rotation you can use key rotation policy, which can be defined on each individual key. 

Our recommendation is to rotate encryption keys at least every two years to meet cryptographic best practices. 

For more information about objects in Key Vault are versioned, see [Key Vault objects, identifiers, and versioning](../general/about-keys-secrets-certificates.md#objects-identifiers-and-versioning). 

## Integarion with Azure services
This feature enables end-to-end zero-touch rotation for encryption at rest for Azure services with customer-managed key (CMK) stored in Azure Key Vault. Please refer to specific Azure service documentation to see if the service covers end-to-end rotation. 

For more information about data encryption in Azure, see:
- [Azure Encryption at Rest](../../security/fundamentals/encryption-atrest.md#azure-encryption-at-rest-components)
- [Azure services data encryption support table](../../security/fundamentals/encryption-models.md#supporting-services)

## Pricing

There's an additional cost per scheduled key rotation. For more information, see [Azure Key Vault pricing page](https://azure.microsoft.com/pricing/details/key-vault/)

## Permissions required

Key Vault key rotation feature requires key management permissions. You can assign a "Key Vault Crypto Officer" role to manage rotation policy and on-demand rotation.

For more information on how to use Key Vault RBAC permission model and assign Azure roles, see [Use an Azure RBAC to control access to keys, certificates and secrets](../general/rbac-guide.md)

> [!NOTE]
> If you use an access policies permission model, it is required to set 'Rotate', 'Set Rotation Policy', and 'Get Rotation Policy' key permissions to manage rotation policy on keys. 

## Key rotation policy

The key rotation policy allows users to configure rotation and Event Grid notifications near expiry notification.

Key rotation policy settings:

-   Expiry time: key expiration interval. It's used to set expiration date on newly rotated key. It doesn't affect a current key.
-   Enabled/disabled: flag to enable or disable rotation for the key
-   Rotation types:
    -   Automatically renew at a given time after creation (default)
    -   Automatically renew at a given time before expiry. It requires 'Expiry Time' set on rotation policy and 'Expiration Date' set on the key.
-   Rotation time: key rotation interval, the minimum value is seven days from creation and seven days from expiration time
-   Notification time: key near expiry event interval for Event Grid notification. It requires 'Expiry Time' set on rotation policy and 'Expiration Date' set on the key. 

> [!IMPORTANT]
> Key rotation generates a new key version of an existing key with new key material. Target services should use versionless key uri to automatically refresh to latest version of the key. Ensure that your data encryption solution stores versioned key uri with data to point to the same key material for decrypt/unwrap as was used for encrypt/wrap operations to avoid disruption to your services. All Azure services are currently following that pattern for data encryption.

:::image type="content" source="../media/keys/key-rotation/key-rotation-1.png" alt-text="Rotation policy configuration":::

## Configure key rotation policy

Configure key rotation policy during key creation.

:::image type="content" source="../media/keys/key-rotation/key-rotation-2.png" alt-text="Configure rotation during key creation":::

Configure rotation policy on existing keys.

:::image type="content" source="../media/keys/key-rotation/key-rotation-3.png" alt-text="Configure rotation on existing key":::

### Azure CLI

Save  key rotation policy to a file. Key rotation policy example:

```json
{
  "lifetimeActions": [
    {
      "trigger": {
        "timeAfterCreate": "P18M",
        "timeBeforeExpiry": null
      },
      "action": {
        "type": "Rotate"
      }
    },
    {
      "trigger": {
        "timeBeforeExpiry": "P30D"
      },
      "action": {
        "type": "Notify"
      }
    }
  ],
  "attributes": {
    "expiryTime": "P2Y"
  }
}
```

Set rotation policy on a key passing previously saved file using Azure CLI [az keyvault key rotation-policy update](/cli/azure/keyvault/key/rotation-policy) command. 

```azurecli
az keyvault key rotation-policy update --vault-name <vault-name> --name <key-name> --value </path/to/policy.json>
```

### Azure PowerShell

Set rotation policy using Azure Powershell [Set-AzKeyVaultKeyRotationPolicy](/powershell/module/az.keyvault/set-azkeyvaultkeyrotationpolicy) cmdlet. 

```powershell
Set-AzKeyVaultKeyRotationPolicy -VaultName <vault-name> -KeyName <key-name> -ExpiresIn (New-TimeSpan -Days 720) -KeyRotationLifetimeAction @{Action="Rotate";TimeAfterCreate= (New-TimeSpan -Days 540)}
```
## Rotation on demand

Key rotation can be invoked manually.

### Portal
Click 'Rotate Now' to invoke rotation.

:::image type="content" source="../media/keys/key-rotation/key-rotation-4.png" alt-text="Rotation on-demand":::

### Azure CLI

Use Azure CLI [az keyvault key rotate](/cli/azure/keyvault/key#az-keyvault-key-rotate) command to rotate key.

```azurecli
az keyvault key rotate --vault-name <vault-name> --name <key-name>
```

### Azure PowerShell

Use Azure PowerShell [Invoke-AzKeyVaultKeyRotation](/powershell/module/az.keyvault/invoke-azkeyvaultkeyrotation) cmdlet.

```powershell
Invoke-AzKeyVaultKeyRotation -VaultName <vault-name> -Name <key-name>
```

## Configure key near expiry notification

Configuration of expiry notification for Event Grid key near expiry event. You can configure notification with days, months and years before expiry to trigger near expiry event. 

:::image type="content" source="../media/keys/key-rotation/key-rotation-5.png" alt-text="Configure Notification":::

For more information about Event Grid notifications in Key Vault, see
[Azure Key Vault as Event Grid source](../../event-grid/event-schema-key-vault.md?tabs=event-grid-event-schema)

## Configure key rotation with ARM template

Key rotation policy can also be configured using ARM templates.

> [!NOTE]
> It requires 'Key Vault Contributor' role on Key Vault configured with Azure RBAC to deploy key through management plane.

```json
{
    "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
        "vaultName": {
            "type": "String",
            "metadata": {
                "description": "The name of the key vault to be created."
            }
        },
        "keyName": {
            "type": "String",
            "metadata": {
                "description": "The name of the key to be created."
            }
        },
        "rotatationTimeAfterCreate": {
            "defaultValue": "P18M",
            "type": "String",
            "metadata": {
                "description": "Time duration to trigger key rotation. i.e. P30D, P1M, P2Y"
            }
        },
        "expiryTime": {
            "defaultValue": "P2Y",
            "type": "String",
            "metadata": {
                "description": "The expiry time for new key version. i.e. P90D, P2M, P3Y"
            }
        },
        "notifyTime": {
            "defaultValue": "P30D",
            "type": "String",
            "metadata": {
                "description": "Near expiry Event Grid notification. i.e. P30D"
            }
        }

    },
    "resources": [
        {
            "type": "Microsoft.KeyVault/vaults/keys",
            "apiVersion": "2021-06-01-preview",
            "name": "[concat(parameters('vaultName'), '/', parameters('keyName'))]",
            "location": "[resourceGroup().location]",
            "properties": {
                "vaultName": "[parameters('vaultName')]",
                "kty": "RSA",
                "rotationPolicy": {
                    "lifetimeActions": [
                        {
                            "trigger": {
                                "timeAfterCreate": "[parameters('rotatationTimeAfterCreate')]",
                                "timeBeforeExpiry": ""
                            },
                            "action": {
                                "type": "Rotate"
                            }
                        },
                        {
                            "trigger": {
                                "timeBeforeExpiry": "[parameters('notifyTime')]"
                            },
                            "action": {
                                "type": "Notify"
                            }
                        }

                    ],
                    "attributes": {
                        "expiryTime": "[parameters('expiryTime')]"
                    }
                }
            }
        }
    ]
}

```

## Resources

- [Monitoring Key Vault with Azure Event Grid](../general/event-grid-overview.md)
- [Use an Azure RBAC to control access to keys, certificates and secrets](../general/rbac-guide.md)
- [Azure Data Encryption At Rest](../../security/fundamentals/encryption-atrest.md)
- [Azure Storage Encryption](../../storage/common/storage-service-encryption.md)
- [Azure Disk Encryption](../../virtual-machines/disk-encryption.md)
