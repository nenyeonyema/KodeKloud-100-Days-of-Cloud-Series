# Azure Blob Lifecycle Management - Task 36

## Overview
This project demonstrates how to automate blob deletion in Azure Storage using **Azure Blob Lifecycle Management**. The objective is to optimize storage costs by automatically deleting blobs that have not been modified for a specified period.
> Task:
> The Nautilus DevOps team needs to optimize data retention costs by automating the deletion of old blobs. They plan to implement Blob Lifecycle Management for a specific container in Azure Storage.
>
> 1) Create a Storage Account:
>
> Name the storage account nautilusstor8796.
> Set the region to East US.
> Use Locally-redundant storage (LRS) as the redundancy option.
> 2) Create a Blob Container:
>
> Name the container nautilus-container8796.
> 3) Upload a File to the Container:
>
> Upload the file named tempfile.txt to the container. The file is present under /root of the client host.
> 4) Configure Blob Lifecycle Management:
>
> Apply a Lifecycle Management rule named nautilus-del-rule to the container nautilus-container8796 to delete blobs after 7 days of last modification.
> 5) Validation:
>
> Verify that the Lifecycle Management rule named nautilus-del-rule is correctly applied.
>
> Use the Azure Portal or Azure CLI for resource creation.
> Ensure the storage account and container are properly configured.


---


## Task Requirements

1. Create an Azure Storage Account:
   - Name: `nautilusstor8796`
   - Region: `East US`
   - Redundancy: `Locally-redundant storage (LRS)`

2. Create a Blob Container:
   - Name: `nautilus-container8796`

3. Upload a file:
   - File: `tempfile.txt`
   - Source location: `/root/tempfile.txt`

4. Configure Blob Lifecycle Management:
   - Rule Name: `nautilus-del-rule`
   - Action: Delete blobs after **7 days** of last modification.

5. Validate that the lifecycle management rule is correctly applied.

---

## Prerequisites

- Azure subscription
- Azure CLI installed and configured
- Access to the `azure-client` host
- Storage account permissions to create resources and management policies

---

## Architecture

```text
Azure Storage Account (nautilusstor8796)
│
├── Blob Container (nautilus-container8796)
│   └── tempfile.txt
│
└── Lifecycle Management Policy
    └── Rule: nautilus-del-rule
        └── Delete blobs after 7 days of last modification
```

---

## Deployment Steps

### 1. Login to Azure

```bash
az login
```

### 2. Identify the Resource Group

```bash
az group list --output table
RG=$(az group list --query "[0].name" -o tsv)
```

### 3. Create the Storage Account

```bash
az storage account create \
  --name nautilusstor8796 \
  --resource-group $RG \
  --location eastus \
  --sku Standard_LRS
```

### 4. Retrieve the Storage Account Key

```bash
ACCOUNT_KEY=$(az storage account keys list \
  --resource-group $RG \
  --account-name nautilusstor8796 \
  --query "[0].value" -o tsv)
```

### 5. Create the Blob Container

```bash
az storage container create \
  --name nautilus-container8796 \
  --account-name nautilusstor8796 \
  --account-key "$ACCOUNT_KEY"
```

### 6. Upload the File

```bash
az storage blob upload \
  --account-name nautilusstor8796 \
  --account-key "$ACCOUNT_KEY" \
  --container-name nautilus-container8796 \
  --name tempfile.txt \
  --file /root/tempfile.txt
```

### 7. Create the Lifecycle Policy File

Create `lifecycle.json`:

```json
{
  "rules": [
    {
      "enabled": true,
      "name": "nautilus-del-rule",
      "type": "Lifecycle",
      "definition": {
        "filters": {
          "blobTypes": [
            "blockBlob"
          ],
          "prefixMatch": [
            "nautilus-container8796/"
          ]
        },
        "actions": {
          "baseBlob": {
            "delete": {
              "daysAfterModificationGreaterThan": 7
            }
          }
        }
      }
    }
  ]
}
```

### 8. Apply the Lifecycle Policy

```bash
az storage account management-policy create \
  --account-name nautilusstor8796 \
  --resource-group $RG \
  --policy @lifecycle.json
```

---

## Verification

Verify that the lifecycle policy has been applied:

```bash
az storage account management-policy show \
  --account-name nautilusstor8796 \
  --resource-group $RG
```

Display only the rule information:

```bash
az storage account management-policy show \
  --account-name nautilusstor8796 \
  --resource-group $RG \
  --query "policy.rules[].{Name:name,Enabled:enabled,DeleteAfter:definition.actions.baseBlob.delete.daysAfterModificationGreaterThan}" \
  -o table
```

Expected output:

```text
Name                Enabled    DeleteAfter
------------------  ---------  -----------
nautilus-del-rule   True       7
```

---

## Key Concepts Learned

- Creating Azure Storage Accounts using Azure CLI
- Working with Azure Blob Containers
- Uploading blobs via Azure CLI
- Configuring Azure Blob Lifecycle Management Policies
- Automating data retention and storage cost optimization
- Validating Azure resources and policies through CLI commands

---

## Outcome

Successfully implemented automated blob lifecycle management in Azure by creating a storage account, uploading blobs to a container, and configuring a lifecycle policy that deletes blobs after seven days of inactivity, thereby improving storage governance and cost efficiency.
