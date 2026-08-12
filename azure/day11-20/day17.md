Markdown
# Azure Task 17 — Create a Public Blob Container

## 📌 Task Overview

As part of the Nautilus DevOps team's data migration to Microsoft Azure, an Azure Storage Account and a public Blob container were required to host assets requiring public distribution. The objective was to provision the storage account using the Azure CLI, explicitly enable public blob access at the account level, create the blob container configured with public read access for its blobs, and verify the access settings via Azure CLI queries.

| Requirement | Value |
| :--- | :--- |
| **Storage Account** | `nautilusst4811` |
| **Blob Container** | `nautilus-blob-28294` |
| **Container Access Level** | Public (`blob`) |
| **Anonymous Access** | Enabled (`true`) |
| **Method** | Azure CLI (`az storage`) |

---

## 🎯 Objective

Establish a publicly accessible blob container within Azure Storage for hosting public data assets:

```text
Authenticate Azure CLI ──► Identify Resource Group ──► Create Storage Account ──► Enable Account Public Access ──► Provision Public Container ──► Verify Access Level
🛠️ Implementation & Verification Commands
Bash
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
  --name nautilusst4811 \
  --resource-group "$RG" \
  --location "$LOCATION" \
  --sku Standard_LRS

# ==========================================
# 4. ENABLE ANONYMOUS BLOB ACCESS AT ACCOUNT LEVEL
# ==========================================
az storage account update \
  --name nautilusst4811 \
  --resource-group "$RG" \
  --allow-blob-public-access true

# ==========================================
# 5. RETRIEVE KEY & PROVISION PUBLIC CONTAINER
# ==========================================
STORAGE_KEY=$(az storage account keys list \
  --resource-group "$RG" \
  --account-name nautilusst4811 \
  --query "[0].value" \
  --output tsv)

az storage container create \
  --name nautilus-blob-28294 \
  --account-name nautilusst4811 \
  --account-key "$STORAGE_KEY" \
  --public-access blob

# ==========================================
# 6. VERIFICATION & INTEGRITY CHECKS
# ==========================================
# Verify Storage Account Public Access Setting (Expected: true)
az storage account show \
  --name nautilusst4811 \
  --resource-group "$RG" \
  --query "allowBlobPublicAccess"

# Verify Container Access Level (Expected: Blob)
az storage container show \
  --name nautilus-blob-28294 \
  --account-name nautilusst4811 \
  --account-key "$STORAGE_KEY" \
  --query "{Name:name, PublicAccess:properties.publicAccess}"
Note: A returned PublicAccess value of "Blob" confirms that anonymous read access is active for blobs inside the nautilus-blob-28294 container.
