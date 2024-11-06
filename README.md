# terraform-vm-win2k22

Virtual machine build, using Azure Storage Account for State backend

## Disclaimer

For learning purposes only. Do not use in production. It is intended for
builds and teardowns of VMs for testing purposes.

## Prerequisites

- Terraform
- Azure CLI
- Azure Subscription

## Pre-configure Azure

### RBAC

#### Key Vault Administrator

Make sure your **admin account** has **KevVault Administrator** role for the subscription.

Azure CLI command to assign the role:

```bash
az role assignment create --role "Key Vault Administrator" --assignee <your-email> --scope /subscriptions/<subscription-id>
```

#### Service Principal for Terraform

Give the **Service Principal** the **Key Vault Secrets User** role for the subscription.

Azure CLI command to assign the role:

```bash
az role assignment create --role "Key Vault Secrets User" --assignee <service-principal-app-id> --scope /subscriptions/<subscription-id>
```

### Build your Storage Container

```bash
#!/bin/bash

RESOURCE_GROUP_NAME=tfstate_rg
STORAGE_ACCOUNT_NAME=tfstate$RANDOM
CONTAINER_NAME=tfstate

# Create resource group
az group create --name $RESOURCE_GROUP_NAME --location norwayeast

# Create storage account
az storage account create --resource-group $RESOURCE_GROUP_NAME --name $STORAGE_ACCOUNT_NAME --sku Standard_LRS --encryption-services blob

# Create blob container
az storage container create --name $CONTAINER_NAME --account-name $STORAGE_ACCOUNT_NAME
```

### Key Vault

```bash
ACCOUNT_KEY=$(az storage account keys list --resource-group $RESOURCE_GROUP_NAME --account-name $STORAGE_ACCOUNT_NAME --query '[0].value' -o tsv)

# Create a Key Vault (if it doesn't already exist)
az keyvault create \
  --name tfstatekv$RANDOM \
  --resource-group $RESOURCE_GROUP_NAME \
  --location <your-location>

# Store the Storage Account access key in Key Vault. Locate <your-keyvault-name> from the previous command or in the Azure Portal.
az keyvault secret set \
  --vault-name <your-keyvault-name> \
  --name tfstatesecret \
  --value "$ACCOUNT_KEY"
```

### Environment Variable

$ARM_ACCESS_KEY is needed for the Terraform backend configuration to work.

```bash
# Set environment variable `ARM_ACCESS_KEY` to the storage account key secret stored in Key Vault:
export ARM_ACCESS_KEY=$(az keyvault secret show \
  --vault-name <your-keyvault-name> \
  --name tfstatesecret \
  --query "value" -o tsv)
```

## Terraform

The **backend** block cannot use variables. It must be hardcoded. Make
sure to locate **storage_account_name** in the Azure Portal as it has
a random number in the end for global uniqueness.

```hcl
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "4.8.0"
    }
  }
  backend "azurerm" {
    resource_group_name  = "tfstate_rg"
    storage_account_name = "tfstateNUMBER"
    container_name       = "tfstate"
    key                  = "terraform.tfstate"
  }
}

provider "azurerm" {
  subscription_id = var.subscription_id
  client_id       = var.client_id
  client_secret   = var.client_secret
  tenant_id       = var.tenant_id
  features {}
}

resource "azurerm_resource_group" "rg" {
  name     = var.rg_name
  location = var.location
}
```
