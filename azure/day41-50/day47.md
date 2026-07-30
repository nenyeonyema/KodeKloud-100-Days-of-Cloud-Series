# Azure Task 47 — Azure SQL Database Backup and Recovery

## Overview

This task demonstrates creating an Azure SQL Database, setting up Blob Storage for backup, exporting the database as a .bacpac file to Blob Storage, and downloading the backup to a local host. This is a common pattern for database migration, disaster recovery preparation, and compliance archiving in cloud environments.

---

## Architecture

```
+------------------------------------------+
|         datacenter-server-26261          |
|         Azure SQL Server                 |
|         West US | v12.0                  |
|         Public Network Access: Enabled   |
|                                          |
|   +------------------------------------+ |
|   |        datacenter-sqldb            | |
|   |        Basic Edition               | |
|   |        2 GB max size               | |
|   |        Local backup redundancy     | |
|   |        Status: Online              | |
|   +------------------------------------+ |
+------------------+-----------------------+
                   |
                   | az sql db export (.bacpac)
                   v
+------------------------------------------+
|         datacenterst29657                |
|         Storage Account                  |
|         West US | Standard_LRS           |
|                                          |
|   +------------------------------------+ |
|   |   datacenter-container-28642       | |
|   |                                    | |
|   |   datacenter-db-backup.bacpac      | |
|   |   2,771 bytes                      | |
|   +------------------------------------+ |
+------------------+-----------------------+
                   |
                   | az storage blob download
                   v
+------------------------------------------+
|         azure-client host                |
|         /opt/datacenter-db-backup.bacpac |
|         2.8K                             |
+------------------------------------------+
```

---

## Prerequisites

- Azure CLI installed and authenticated
- Resource group: kml_rg_main-f2da7b99310c4f39
- Region: West US

---

## Task 1 — Create Azure SQL Database

### Step 1 — Login

```bash
az login -u kk_lab_user_main-f2da7b99310c4f39@azurefreekmlprod.onmicrosoft.com -p "a#t7@p84"
az group list --query "[0].name" -o tsv
```

### Step 2 — Create SQL Server

```bash
az sql server create \
  --name datacenter-server-26261 \
  --resource-group kml_rg_main-f2da7b99310c4f39 \
  --location westus \
  --admin-user datacenter-admin \
  --admin-password "D@taC3nter#Secure2026!"
```

**Password requirements:** Azure SQL Server passwords must meet complexity requirements — uppercase, lowercase, numbers, and special characters. Simple passwords like `DataCenter@123456` were rejected; `D@taC3nter#Secure2026!` satisfied the policy.

### Step 3 — Allow Public Access via Firewall Rule

```bash
az sql server firewall-rule create \
  --resource-group kml_rg_main-f2da7b99310c4f39 \
  --server datacenter-server-26261 \
  --name AllowAllIPs \
  --start-ip-address 0.0.0.0 \
  --end-ip-address 255.255.255.255
```

This opens the SQL Server to all IP addresses — required for public accessibility. For production, restrict to specific IP ranges.

### Step 4 — Create the SQL Database

```bash
az sql db create \
  --resource-group kml_rg_main-f2da7b99310c4f39 \
  --server datacenter-server-26261 \
  --name datacenter-sqldb \
  --edition Basic \
  --capacity 5 \
  --max-size 2GB \
  --backup-storage-redundancy Local
```

**Flags explained:**
- `--edition Basic` — lightweight tier for less demanding workloads
- `--capacity 5` — 5 DTUs (Basic tier unit)
- `--max-size 2GB` — sets database size limit to 2 GiB
- `--backup-storage-redundancy Local` — locally-redundant backup storage as required

### Step 5 — Verify Database is Online

```bash
az sql db show \
  --resource-group kml_rg_main-f2da7b99310c4f39 \
  --server datacenter-server-26261 \
  --name datacenter-sqldb \
  --query "{name:name, status:status, edition:edition, maxSizeBytes:maxSizeBytes}" \
  -o table
```

Output:
```
Name              Status    Edition    MaxSizeBytes
----------------  --------  ---------  --------------
datacenter-sqldb  Online    Basic      2147483648
```

---

## Task 2 — Create Storage Account and Blob Container

### Step 6 — Create Storage Account

```bash
az storage account create \
  --name datacenterst29657 \
  --resource-group kml_rg_main-f2da7b99310c4f39 \
  --location westus \
  --sku Standard_LRS
```

### Step 7 — Create Blob Container

```bash
az storage container create \
  --name datacenter-container-28642 \
  --account-name datacenterst29657 \
  --auth-mode login
```

### Step 8 — Get Storage Account Key

```bash
az storage account keys list \
  --account-name datacenterst29657 \
  --resource-group kml_rg_main-f2da7b99310c4f39 \
  --query "[0].value" -o tsv
```

---

## Task 3 — Export Database Backup to Blob Storage

```bash
az sql db export \
  --resource-group kml_rg_main-f2da7b99310c4f39 \
  --server datacenter-server-26261 \
  --name datacenter-sqldb \
  --admin-user datacenter-admin \
  --admin-password "D@taC3nter#Secure2026!" \
  --storage-key "<STORAGE_KEY>" \
  --storage-key-type StorageAccessKey \
  --storage-uri "https://datacenterst29657.blob.core.windows.net/datacenter-container-28642/datacenter-db-backup.bacpac"
```

Export result:
```json
{
  "blobUri": "https://datacenterst29657.blob.core.windows.net/datacenter-container-28642/datacenter-db-backup.bacpac",
  "databaseName": "datacenter-sqldb",
  "requestType": "ExportDatabase",
  "status": "Completed",
  "queuedTime": "7/30/2026 11:34:09 AM",
  "lastModifiedTime": "7/30/2026 11:38:12 AM"
}
```

Export completed in approximately 4 minutes.

---

## Task 4 — Download Backup to /opt

```bash
az storage blob download \
  --account-name datacenterst29657 \
  --account-key "<STORAGE_KEY>" \
  --container-name datacenter-container-28642 \
  --name datacenter-db-backup.bacpac \
  --file /opt/datacenter-db-backup.bacpac
```

Verify:
```bash
ls -lh /opt/datacenter-db-backup.bacpac
```

Output:
```
-rw-r--r-- 1 root root 2.8K Jul 30 11:40 /opt/datacenter-db-backup.bacpac
```

---

## Key Concepts

**What is a .bacpac file?**
A .bacpac file is Azure SQL Database's portable export format. It contains both the database schema (tables, indexes, constraints) and the data. It is used for:
- Migrating databases between servers or subscriptions
- Archiving point-in-time snapshots
- Moving databases between Azure and on-premises SQL Server

It differs from a .bak file (SQL Server native backup) — .bacpac is schema + data in a ZIP-based format, .bak is a binary backup specific to SQL Server.

**What is a DTU?**
DTU (Database Transaction Unit) is Azure SQL's blended performance metric combining CPU, memory, and I/O. Basic tier provides 5 DTUs — sufficient for dev/test and light workloads. For production, Standard or Premium tiers with more DTUs or vCore-based pricing are used.

**Backup Storage Redundancy options**

| Option | Description | Use case |
|---|---|---|
| Local (LRS) | 3 copies in one datacenter | Cost-sensitive, non-critical |
| Zone (ZRS) | 3 copies across availability zones | Higher availability |
| Geo (GRS) | Copies in paired region | Disaster recovery |

This task used Local redundancy as specified.

**Why export to .bacpac instead of using built-in backups?**
Azure SQL Database has automatic built-in backups (full, differential, log) but they are managed by Azure and not directly downloadable as files. The `az sql db export` command creates a portable .bacpac that can be stored anywhere, downloaded, and imported into any SQL Server instance — making it ideal for migration and off-platform archiving.

**SQL Server vs SQL Database**
Azure SQL has two levels:

| Resource | Role |
|---|---|
| SQL Server | The logical container — holds credentials, firewall rules, location |
| SQL Database | The actual database — holds data, edition, size settings |

You must create the server first, then the database inside it. Multiple databases can share one server.

---

## Lessons Learned

1. SQL Server admin passwords must meet complexity requirements — include uppercase, lowercase, numbers, and special characters.
2. The firewall rule with `0.0.0.0` to `255.255.255.255` is required for public access — without it, export operations fail because Azure cannot reach the server.
3. `az sql db export` is asynchronous but the CLI waits for completion by default — status `Completed` in the response confirms the backup is fully written to Blob Storage.
4. The .bacpac file extension must be included in the `--storage-uri` — it is not added automatically.

---

## Outcome

| Task | Result |
|---|---|
| SQL Server datacenter-server-26261 created (West US) | Done |
| Firewall rule added for public access | Done |
| datacenter-sqldb created (Basic, 2GB, Local redundancy) | Done |
| DB status: Online | Done |
| Storage account datacenterst29657 created (Standard_LRS) | Done |
| Blob container datacenter-container-28642 created | Done |
| Database exported to datacenter-db-backup.bacpac | Done |
| Export status: Completed (2,771 bytes) | Done |
| Backup downloaded to /opt/datacenter-db-backup.bacpac (2.8K) | Done |
