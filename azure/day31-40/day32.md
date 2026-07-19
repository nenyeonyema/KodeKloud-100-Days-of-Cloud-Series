# Azure Task 32 — Blob Container Data Migration

## Overview

This task demonstrates migrating data between two Azure Blob containers within the same storage account. A new private destination container is created, a file is copied from the source container, and data consistency is verified by comparing file content in both containers. This is a common pattern in cloud data migration and storage reorganization workflows.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    datacenterst19735                        │
│                    Storage Account                          │
│                                                             │
│   ┌─────────────────────────┐   ┌────────────────────────┐  │
│   │  datacenter-source-2453 │   │  datacenter-dest-11288 │  │
│   │  (existing container)   │   │  (new private container│  │
│   │                         │   │                        │  │
│   │  datacenter.txt ────────┼──►│  datacenter.txt        │  │
│   │                         │   │                        │  │
│   └─────────────────────────┘   └────────────────────────┘  │
│                                                             │
│         az storage blob copy start                          │
└─────────────────────────────────────────────────────────────┘
```

---

## Prerequisites

- Azure CLI installed and authenticated
- Storage account `datacenterst19735` already exists
- Source container `datacenter-source-2453` with `datacenter.txt` already exists
- Resource group: `kml_rg_main-4938febae8fb4b31`

---

## Step 1 — Login

```bash
az login -u kk_lab_user_main-4938febae8fb4b31@azurefreekmlprod.onmicrosoft.com -p "s^sssr86"
```

---

## Step 2 — Create the Destination Container (Private)

```bash
az storage container create \
  --name datacenter-dest-11288 \
  --account-name datacenterst19735 \
  --auth-mode login \
  --public-access off
```

Expected output:
```json
{ "created": true }
```

**`--public-access off`** ensures the new container is private — no anonymous access to blobs inside it.

---

## Step 3 — Copy the Blob from Source to Destination

```bash
az storage blob copy start \
  --account-name datacenterst19735 \
  --destination-container datacenter-dest-11288 \
  --destination-blob datacenter.txt \
  --source-container datacenter-source-2453 \
  --source-blob datacenter.txt \
  --auth-mode login
```

This initiates a **server-side copy** — Azure copies the blob directly between containers without the data passing through the client machine, making it faster and more efficient than download + re-upload.

---

## Step 4 — Check Copy Status

```bash
az storage blob show \
  --account-name datacenterst19735 \
  --container-name datacenter-dest-11288 \
  --name datacenter.txt \
  --auth-mode login \
  --query "properties.copy.status" \
  -o tsv
```

Expected output: `success`

---

## Step 5 — Verify File Exists in Both Containers

```bash
# Source container
az storage blob list \
  --account-name datacenterst19735 \
  --container-name datacenter-source-2453 \
  --auth-mode login \
  --query "[].{name:name, size:properties.contentLength}" \
  -o table

# Destination container
az storage blob list \
  --account-name datacenterst19735 \
  --container-name datacenter-dest-11288 \
  --auth-mode login \
  --query "[].{name:name, size:properties.contentLength}" \
  -o table
```

Both should show `datacenter.txt` with the same file size.

---

## Step 6 — Verify Content is Identical

```bash
# Download from source
az storage blob download \
  --account-name datacenterst19735 \
  --container-name datacenter-source-2453 \
  --name datacenter.txt \
  --file /tmp/source.txt \
  --auth-mode login

# Download from destination
az storage blob download \
  --account-name datacenterst19735 \
  --container-name datacenter-dest-11288 \
  --name datacenter.txt \
  --file /tmp/dest.txt \
  --auth-mode login

# Compare
diff /tmp/source.txt /tmp/dest.txt && echo "Files are identical ✅" || echo "Files differ ❌"
```

---

## Key Concepts

**Server-side copy vs download + re-upload**
`az storage blob copy start` performs a server-side copy entirely within Azure's infrastructure. The data never leaves Azure's network — no bandwidth is consumed on the client side and the operation is significantly faster for large files.

| Method | How it works | Best for |
|---|---|---|
| `blob copy start` | Azure copies blob internally | Same or cross-account copies |
| Download + upload | Data passes through client | Cross-cloud or local migrations |

**Why verify content after copy?**
File size matching confirms the blob was transferred completely. A `diff` on downloaded copies confirms byte-level integrity — important in data migration workflows where silent corruption must be ruled out.

**Private container (`--public-access off`)**
Setting public access to off at the container level means blobs can only be accessed via authenticated requests (account key, SAS token, or Azure AD). This is the correct default for sensitive or internal data.

**`--auth-mode login` vs `--account-key`**
`--auth-mode login` uses the currently logged-in Azure AD identity for authentication — cleaner and more secure than passing account keys directly in commands. Use `--account-key` as a fallback when Azure AD RBAC permissions are not configured.

---

## Outcome

| Task | Result |
|---|---|
| Destination container datacenter-dest-11288 created (private) | ✅ |
| datacenter.txt copied from source to destination | ✅ |
| Copy status → success | ✅ |
| File exists in both containers with matching size | ✅ |
| Content diff → Files are identical | ✅ |
