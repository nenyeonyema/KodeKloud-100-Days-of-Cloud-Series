# Azure Task 38 — VM to Blob Storage Integration

## Overview

This task demonstrates setting up an Azure Virtual Machine to interact with Azure Blob Storage for storing and retrieving data. It covers creating a private storage account, configuring a Blob container, and uploading a file from a VM to Blob Storage using the Azure CLI.

---

## Architecture

```
┌─────────────────┐        scp         ┌─────────────────┐
│    Lab Host     │◄───────────────────│  datacenter-vm  │
│  (azure-client) │                    │  East US        │
│                 │                    │  13.92.35.212   │
│  az storage     │                    │                 │
│  blob upload    │                    │  testfile.txt   │
└────────┬────────┘                    └─────────────────┘
         │
         │ HTTPS (account key auth)
         ▼
┌─────────────────────────────────────┐
│        datacenterstor23588          │
│        Storage Account (LRS)        │
│        East US | Private            │
│                                     │
│  ┌──────────────────────────────┐   │
│  │  datacenter-container23588   │   │
│  │  (private blob container)    │   │
│  │                              │   │
│  │  testfile.txt  (20 bytes)    │   │
│  └──────────────────────────────┘   │
└─────────────────────────────────────┘
```

---

## Prerequisites

- Azure CLI installed and authenticated
- `datacenter-vm` already exists in East US
- Resource group: `kml_rg_main-c6dbd4f9b15a48df`

---

## Step 1 — Login

```bash
az login -u kk_lab_user_main-c6dbd4f9b15a48df@azurefreekmlprod.onmicrosoft.com -p "XvWT8C73"
```

---

## Step 2 — Create the Storage Account

```bash
az storage account create \
  --name datacenterstor23588 \
  --resource-group kml_rg_main-c6dbd4f9b15a48df \
  --location eastus \
  --sku Standard_LRS \
  --allow-blob-public-access false
```

**Flags explained:**
- `--sku Standard_LRS` — Locally-redundant storage, replicates data 3x within a single datacenter
- `--allow-blob-public-access false` — Ensures the storage account is private; no anonymous access to blobs

---

## Step 3 — Create the Private Blob Container

```bash
az storage container create \
  --name datacenter-container23588 \
  --account-name datacenterstor23588 \
  --auth-mode login \
  --public-access off
```

Expected output:
```json
{ "created": true }
```

---

## Step 4 — Retrieve the Storage Account Access Key

```bash
az storage account keys list \
  --account-name datacenterstor23588 \
  --resource-group kml_rg_main-c6dbd4f9b15a48df \
  --query "[0].value" -o tsv
```

This key is used to authenticate blob upload/download operations.

---

## Step 5 — Get the VM's Public IP

```bash
az vm list-ip-addresses \
  --resource-group kml_rg_main-c6dbd4f9b15a48df \
  --name datacenter-vm \
  --query "[].virtualMachine.network.publicIpAddresses[0].ipAddress" -o tsv
```

Output: `13.92.35.212`

---

## Step 6 — Create the Test File on the VM

SSH into the VM and create the file:

```bash
ssh azureuser@13.92.35.212
```

```bash
echo "this is a test file" > /home/azureuser/testfile.txt
cat /home/azureuser/testfile.txt
```

Expected output:
```
this is a test file
```

Then exit the VM:
```bash
exit
```

---

## Step 7 — Copy the File to the Lab Host

The `az storage blob upload` command references a **local** file path on the machine running the command. Since the file exists on the VM, copy it to the lab host first:

```bash
scp azureuser@13.92.35.212:/home/azureuser/testfile.txt /root/testfile.txt
```

Expected output:
```
testfile.txt    100%   20     0.2KB/s   00:00
```

---

## Step 8 — Upload the File to Blob Storage

```bash
az storage blob upload \
  --account-name datacenterstor23588 \
  --account-key "<ACCESS_KEY>" \
  --container-name datacenter-container23588 \
  --name testfile.txt \
  --file /root/testfile.txt
```

Expected output:
```json
{
  "lastModified": "2026-07-07T10:48:34+00:00",
  "request_server_encrypted": true,
  "version": "2022-11-02"
}
```

---

## Step 9 — Verify the Upload

```bash
az storage blob list \
  --account-name datacenterstor23588 \
  --account-key "<ACCESS_KEY>" \
  --container-name datacenter-container23588 \
  --query "[].{name:name, size:properties.contentLength}" \
  -o table
```

Expected output:
```
Name          Size
------------  ------
testfile.txt  20
```

---

## Key Concepts

**Why `--allow-blob-public-access false`?**
By default, Azure storage accounts may allow public blob access if containers are configured that way. Setting this at the account level enforces privacy regardless of individual container settings — a security best practice.

**Why `Standard_LRS`?**
Locally-redundant storage keeps 3 synchronous copies of data within a single Azure datacenter. It is the most cost-effective redundancy option, suitable for non-critical or dev/test workloads.

**Why copy the file to the lab host before uploading?**
The `az storage blob upload --file` flag takes a path on the local machine running the command. The file existed on the remote VM, not the lab host. Using `scp` to pull the file first avoids needing to install and configure the Azure CLI on the VM itself.

**Why use account key authentication?**
Account key auth is straightforward for scripted uploads and gives full access to the storage account. For production workloads, Managed Identity or SAS tokens with scoped permissions are preferred over account keys.

---

## Outcome

| Task | Result |
|---|---|
| Storage account created (East US, LRS, private) | ✅ |
| Blob container created (private, no public access) | ✅ |
| Access key retrieved | ✅ |
| testfile.txt created on VM with correct content | ✅ |
| File copied from VM to lab host via scp | ✅ |
| File uploaded to blob container | ✅ |
| Blob verified in container (20 bytes) | ✅ |
