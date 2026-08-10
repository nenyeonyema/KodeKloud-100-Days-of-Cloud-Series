# Task 19 - Convert a Public Azure Blob Container to Private

## Overview

This task is part of the Nautilus DevOps team's cloud management operations on Microsoft Azure. The team identified that one of their Azure Blob Storage containers was incorrectly set to public access, posing a potential security risk. This task focuses on converting the public blob container `xfusion-container-4584` to private while leaving the other container `xfusion-priv-25262` unchanged.

---

## Objectives

- Convert the blob container `xfusion-container-4584` from public to private
- Ensure the access level for `xfusion-container-4584` is set to `private` with no public access
- Leave the container `xfusion-priv-25262` unchanged

---

## Prerequisites

- Access to an active Azure subscription or lab credentials
- Azure CLI installed and authenticated
- An existing Resource Group
- An existing storage account named `xfusionst19295` in the `westus` region
- Two existing blob containers:
  - `xfusion-container-4584` (currently public)
  - `xfusion-priv-25262` (already private)

---

## Tools Used

| Tool | Purpose |
|------|---------|
| Azure CLI | Managing blob container access via command line |
| Microsoft Azure | Cloud platform |

---

## Steps

### 1. Authenticate with Azure

```bash
az login
```

> For KodeKloud labs, credentials are pre-configured. Skip this step if already logged in.

### 2. Identify the Resource Group

```bash
az group list --output table
```

### 3. Verify Current Access Levels of Both Containers

```bash
az storage container list \
  --account-name xfusionst19295 \
  --query "[].{Name:name, PublicAccess:properties.publicAccess}" \
  --output table
```

Confirm that `xfusion-container-4584` is currently public before making changes.

### 4. Convert the Container from Public to Private

```bash
az storage container set-permission \
  --name xfusion-container-4584 \
  --account-name xfusionst19295 \
  --public-access off
```

### 5. Verify the Access Level Change

```bash
az storage container list \
  --account-name xfusionst19295 \
  --query "[].{Name:name, PublicAccess:properties.publicAccess}" \
  --output table
```

---

## Expected Output

**Before change:**
```
Name                      PublicAccess
------------------------  ------------
xfusion-container-4584    blob
xfusion-priv-25262        None
```

**After change:**
```
Name                      PublicAccess
------------------------  ------------
xfusion-container-4584    None
xfusion-priv-25262        None
```

---

## Blob Container Access Levels

| Access Level | CLI Value | Description |
|--------------|-----------|-------------|
| Private | `off` | No public access. Only the storage account owner can access the container and its blobs. |
| Blob | `blob` | Public read access for blobs only. Container metadata and list of blobs are not publicly accessible. |
| Container | `container` | Full public read access. Anyone can list and read the container and all its blobs. |

---

## Key Concepts

- **Azure Blob Storage:** Microsoft's object storage solution for the cloud, optimized for storing massive amounts of unstructured data such as text, images, videos, and backups.
- **Blob Container:** A logical grouping of blobs within a storage account, similar to a directory. Access control is set at the container level.
- **Public Access Level:** Determines whether blobs in a container are accessible anonymously over the internet without authentication.
- **Private Access:** The most restrictive and recommended setting for production data. Requires authentication via storage account keys, SAS tokens, or Azure AD to access any blob.
- **Storage Account:** The top-level Azure resource that provides a unique namespace for all Azure Storage data. A storage account can contain multiple containers.

---

## Notes

- Always verify the current access level before and after making changes to confirm the operation was successful.
- Setting a container to private does **not** affect existing Shared Access Signatures (SAS tokens) — those remain valid until they expire.
- If the storage account has `AllowBlobPublicAccess` disabled at the account level, no container within it can be set to public regardless of container-level settings.
- To disable public access at the storage account level entirely (more secure):
  ```bash
  az storage account update \
    --name xfusionst19295 \
    --resource-group <ResourceGroupName> \
    --allow-blob-public-access false
  ```
- In production, always follow the **principle of least privilege** — containers should be private by default and only made public when explicitly required.

---

