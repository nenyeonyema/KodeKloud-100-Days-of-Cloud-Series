# Azure Task 42 — Blob Container Cleanup: Download and Delete

## Overview

This task demonstrates an Azure environment cleanup operation. The contents of a private blob container are downloaded to a local directory on the azure-client host, and the container is then deleted from the storage account. This is a common pattern in cloud migrations where resources are created for one-time use and must be decommissioned after data retrieval.

---

## Architecture

```
┌─────────────────────────────────────────────┐
│           nautilusst14495                   │
│           Storage Account                  │
│           West US                           │
│                                             │
│   ┌─────────────────────────────────────┐   │
│   │     nautilus-blob-13597             │   │
│   │     (private blob container)        │   │
│   │                                     │   │
│   │     nautilus.txt  (33 bytes)        │   │
│   └─────────────────────────────────────┘   │
└──────────────────┬──────────────────────────┘
                   │
                   │ az storage blob download-batch
                   ▼
        ┌─────────────────────┐
        │   azure-client host │
        │   /opt/             │
        │   nautilus.txt ✅   │
        └─────────────────────┘
                   │
                   │ az storage container delete
                   ▼
        nautilus-blob-13597 → deleted ✅
```

---

## Prerequisites

- Azure CLI installed and authenticated
- Resource group: `kml_rg_main-66efc41fbfea4105`
- Storage account `nautilusst14495` and container `nautilus-blob-13597` already exist in West US

---

## Step 1 — Login

```bash
az login -u kk_lab_user_main-66efc41fbfea4105@azurefreekmlprod.onmicrosoft.com -p "sD#%cesp"
```

---

## Step 2 — Get Resource Group

```bash
az group list --query "[0].name" -o tsv
```

Output: `kml_rg_main-66efc41fbfea4105`

---

## Step 3 — Get the Storage Account Key

```bash
az storage account keys list \
  --account-name nautilusst14495 \
  --resource-group kml_rg_main-66efc41fbfea4105 \
  --query "[0].value" -o tsv
```

This key authenticates all subsequent blob and container operations.

---

## Step 4 — List Blobs in the Container

```bash
az storage blob list \
  --account-name nautilusst14495 \
  --account-key <KEY> \
  --container-name nautilus-blob-13597 \
  --query "[].{name:name, size:properties.contentLength}" \
  -o table
```

Output:
```
Name          Size
------------  ------
nautilus.txt  33
```

Always list blobs before downloading — confirms what exists and avoids surprises during batch operations.

---

## Step 5 — Download All Blobs to /opt

```bash
az storage blob download-batch \
  --account-name nautilusst14495 \
  --account-key <KEY> \
  --source nautilus-blob-13597 \
  --destination /opt
```

Output:
```
Finished[#############################################################]  100.0000%
[
  "nautilus.txt"
]
```

`download-batch` downloads all blobs in a container in one command, preserving the blob name as the local filename. It is the correct tool for bulk downloads vs `az storage blob download` which handles single files.

---

## Step 6 — Verify Files in /opt

```bash
ls -lh /opt
```

Output:
```
-rw-r--r-- 1 root root  33 Jul 16 10:37 nautilus.txt
```

✅ `nautilus.txt` successfully copied to `/opt`.

---

## Step 7 — Delete the Blob Container

```bash
az storage container delete \
  --account-name nautilusst14495 \
  --account-key <KEY> \
  --name nautilus-blob-13597
```

Output:
```json
{ "deleted": true }
```

---

## Step 8 — Verify Container is Gone

```bash
az storage container list \
  --account-name nautilusst14495 \
  --account-key <KEY> \
  --query "[].name" -o table
```

Output: *(empty — no containers remain)*

✅ Container successfully deleted from the storage account.

---

## Key Concepts

**`download-batch` vs `download`**

| Command | Use case |
|---|---|
| `az storage blob download` | Single blob download |
| `az storage blob download-batch` | All blobs in a container in one operation |

For cleanup tasks involving multiple files, `download-batch` is the right tool — it handles the entire container in one command without needing to loop over individual blobs.

**Why list blobs before downloading?**
Listing first confirms the container is not empty and shows exactly what will be downloaded. In cleanup operations, this acts as an audit step before any destructive actions (deletion) are taken.

**Why delete the container and not just the blobs?**
Deleting the container removes both the blobs and the container itself, fully cleaning up the resource from the storage account. Deleting blobs individually would leave an empty container behind, which still counts as a resource.

**Private container access**
The container `nautilus-blob-13597` is private, meaning blobs inside it are not publicly accessible via URL. Authentication via the storage account key is required for all read and write operations.

---

## Outcome

| Task | Result |
|---|---|
| Storage account key retrieved | ✅ |
| Blob listed → nautilus.txt (33 bytes) | ✅ |
| nautilus.txt downloaded to /opt | ✅ |
| nautilus-blob-13597 container deleted | ✅ |
| Container list returns empty | ✅ |
