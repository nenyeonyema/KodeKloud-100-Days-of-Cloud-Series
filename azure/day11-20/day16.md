# Azure Task 16 — Create a Private Blob Container

## 📌 Task Overview

As part of the Nautilus DevOps team's data migration to Microsoft Azure, a dedicated Azure Storage Account and a private Blob container were required for storing enterprise data securely. The goal was to provision the storage account using the Azure CLI, create the blob container, enforce strict private access controls, and verify that public anonymous access remains completely disabled.

| Requirement | Value |
| :--- | :--- |
| **Storage Account** | `datacenterst26303` |
| **Blob Container** | `datacenter-blob-10124` |
| **Container Access Level** | Private (`off`) |
| **Public Access** | Disabled (`null`) |
| **Method** | Azure CLI (`az storage`) |

---

## 🎯 Objective

Establish a secure, private cloud storage destination within Azure Storage:

```text
Authenticate Azure CLI ──► Identify Resource Group ──► Create Storage Account ──► Provision Private Container ──► Verify Access Level

Implementation & Verification Commands
# ==========================================
# 1. RETRIEVE CREDENTIALS & AUTHENTICATE
# ==========================================
showcreds
az login
az account show --output table

# ==========================================
# 2. IDENTIFY RESOURCE GROUP & LOCATION
# ==========================================
RG=$(az group list --query "[?contains(name, 'kml')].name | [0]" --output tsv)
LOCATION=$(az group show --name "$RG" --query "location" --output tsv)
echo "Resource Group: $RG \vert{} Location:$LOCATION"

# ==========================================
# 3. CREATE STORAGE ACCOUNT
# ==========================================
az storage account create \
  --name datacenterst26303 \
  --resource-group "$RG" \
  --location "$LOCATION" \
  --sku Standard_LRS

# ==========================================
# 4. PROVISION PRIVATE BLOB CONTAINER
# ==========================================
STORAGE_KEY=$(az storage account keys list \
  --resource-group "$RG" \
  --account-name datacenterst26303 \
  --query "[0].value" \
  --output tsv)

az storage container create \
  --name datacenter-blob-10124 \
  --account-name datacenterst26303 \
  --account-key "$STORAGE_KEY" \
  --public-access off

# ==========================================
# 5. VERIFICATION & INTEGRITY CHECKS
# ==========================================
# Verify Storage Account Status
az storage account show \
  --resource-group "$RG" \
  --name datacenterst26303 \
  --query "{Name:name, Location:location, SKU:sku.name, ProvisioningState:provisioningState}" \
  --output table

# Verify Container Creation
az storage container show \
  --name datacenter-blob-10124 \
  --account-name datacenterst26303 \
  --account-key "$STORAGE_KEY"

# Confirm Public Access is Disabled (Expected output: null)
az storage container show \
  --name datacenter-blob-10124 \
  --account-name datacenterst26303 \
  --account-key "$STORAGE_KEY" \
  --query "properties.publicAccess"
