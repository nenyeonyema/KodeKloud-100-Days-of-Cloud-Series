# Azure Task 18 — Upload a File to an Azure Blob Container

## 📌 Task Overview

As part of the Nautilus DevOps team's data migration to Microsoft Azure, a file was required to be transferred from the Azure client host into an existing Azure Blob Storage container. The objective was to locate the local file `/tmp/nautilus.txt`, retrieve access credentials for the pre-existing storage account, upload the file using the Azure CLI, and verify the uploaded blob within the target container.

| Requirement | Value |
| :--- | :--- |
| **Storage Account** | `nautilusst9149` |
| **Region** | `westus` |
| **Blob Container** | `nautilus-blob-18039` |
| **Source File** | `/tmp/nautilus.txt` |
| **Target Blob Name** | `nautilus.txt` |
| **Method** | Azure CLI (`az storage blob`) |

---

## 🎯 Objective

Transfer a local filesystem asset into an existing Azure Blob Storage container:

```text
Authenticate Azure CLI ──► Verify Source File ──► Retrieve Storage Key ──► Upload File via 'az storage blob upload' ──► Verify Blob Presence
🛠️ Implementation & Verification Commands
Bash
# ==========================================
# 1. RETRIEVE CREDENTIALS & AUTHENTICATE
# ==========================================
showcreds
az login
az account show --output table

# ==========================================
# 2. IDENTIFY RESOURCE GROUP & VERIFY SOURCE
# ==========================================
RG=$(az group list --query "[?contains(name, 'kml')].name | [0]" --output tsv)
echo "Target Resource Group: $RG"

# Verify source file presence
ls -la /tmp/nautilus.txt

# ==========================================
# 3. RETRIEVE KEY & VERIFY CONTAINER
# ==========================================
STORAGE_KEY=$(az storage account keys list \
  --resource-group "$RG" \
  --account-name nautilusst9149 \
  --query "[0].value" \
  --output tsv)

az storage container show \
  --name nautilus-blob-18039 \
  --account-name nautilusst9149 \
  --account-key "$STORAGE_KEY"

# ==========================================
# 4. UPLOAD FILE TO BLOB STORAGE
# ==========================================
az storage blob upload \
  --account-name nautilusst9149 \
  --account-key "$STORAGE_KEY" \
  --container-name nautilus-blob-18039 \
  --name nautilus.txt \
  --file /tmp/nautilus.txt

# ==========================================
# 5. VERIFICATION & INTEGRITY CHECKS
# ==========================================
# List container contents to verify uploaded blob
az storage blob list \
  --account-name nautilusst9149 \
  --account-key "$STORAGE_KEY" \
  --container-name nautilus-blob-18039 \
  --output table

# Show detailed blob properties
az storage blob show \
  --account-name nautilusst9149 \
  --account-key "$STORAGE_KEY" \
  --container-name nautilus-blob-18039 \
  --name nautilus.txt \
  --query "{Name:name, Size:properties.contentLength, ContentType:properties.contentType, LastModified:properties.lastModified}" \
  --output table
Note: Verifying blob properties with az storage blob show confirms successful object creation, size parity, and proper metadata upload.
