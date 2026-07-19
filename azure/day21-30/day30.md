# Task 30 — Create Azure SQL Database Instance

## Overview

The Nautilus DevOps Team is migrating part of their infrastructure to Azure. As part of this incremental migration, a publicly accessible Azure SQL Database instance was created with specific compute, storage, backup, and sizing configurations. The database is named `devops-sqldb` and is hosted on a SQL server named `devops-server-4878` in the South Central US region.

---

## Requirements

| Parameter | Value |
|---|---|
| Database Name | `devops-sqldb` |
| Server Name | `devops-server-4878` |
| Region | South Central US |
| Compute + Storage | Basic (for less demanding workloads) |
| Backup Storage Redundancy | Locally-redundant (`Local`) |
| Admin Username | `devops-admin` |
| Admin Password | Strong password (meeting Azure policy) |
| Database Size | 2 GiB |
| Public Access | Yes — accessible from internet |
| Final State | Ready |

---

## Prerequisites

- Azure CLI installed and configured on the azure-client host
- Access to Azure credentials via `showcreds` command
- Resource group already exists in the subscription

---

## Steps Performed

### 1. Login to Azure
```bash
az login -u "<username>" -p "<password>"
az group list --output table
```

### 2. Create the SQL Server
```bash
az sql server create \
    --resource-group kml_rg_main-c81c42ae9dd0406d \
    --name devops-server-4878 \
    --location southcentralus \
    --admin-user devops-admin \
    --admin-password 'N@utiLus#Str0ng!2026'

echo "SQL Server created!"
```

> **Note:** Azure SQL Server password must meet strict complexity requirements — uppercase, lowercase, numbers, and special characters. The username (`devops-admin`) must not appear in the password.

### 3. Allow Public Internet Access via Firewall Rule
```bash
az sql server firewall-rule create \
    --resource-group kml_rg_main-c81c42ae9dd0406d \
    --server devops-server-4878 \
    --name allow-all-ips \
    --start-ip-address 0.0.0.0 \
    --end-ip-address 255.255.255.255

echo "Firewall rule added!"
```

> Setting IP range `0.0.0.0` to `255.255.255.255` makes the database publicly accessible from any IP address — satisfying the "publicly accessible" requirement.

### 4. Create the SQL Database
```bash
az sql db create \
    --resource-group kml_rg_main-c81c42ae9dd0406d \
    --server devops-server-4878 \
    --name devops-sqldb \
    --edition Basic \
    --capacity 5 \
    --max-size 2GB \
    --backup-storage-redundancy Local \
    --service-objective Basic

echo "SQL Database created!"
```

**Flags explained:**
| Flag | Value | Reason |
|---|---|---|
| `--edition Basic` | Basic | Matches "Basic — for less demanding workloads" requirement |
| `--capacity 5` | 5 DTUs | Basic tier uses DTUs; 5 is the only capacity available for Basic |
| `--max-size 2GB` | 2 GiB | Sets database size to 2 GiB as required |
| `--backup-storage-redundancy Local` | Local | Locally-redundant backup storage as required |
| `--service-objective Basic` | Basic | Sets the performance tier to Basic |

### 5. Verify Database is in Ready State
```bash
az sql db show \
    --resource-group kml_rg_main-c81c42ae9dd0406d \
    --server devops-server-4878 \
    --name devops-sqldb \
    --query "{Name:name, Status:status, Edition:edition, MaxSizeBytes:maxSizeBytes, BackupRedundancy:requestedBackupStorageRedundancy}" \
    -o table
```

### 6. Verify Server is Publicly Accessible
```bash
az sql server show \
    --resource-group kml_rg_main-c81c42ae9dd0406d \
    --name devops-server-4878 \
    --query "{Name:name, FQDN:fullyQualifiedDomainName, PublicAccess:publicNetworkAccess, AdminLogin:administratorLogin}" \
    -o table
```

---

## Issues Encountered & Fixes

| Issue | Cause | Fix |
|---|---|---|
| `PasswordNotComplex` on first attempt | Password `DevOps@Admin#2026` did not meet Azure SQL complexity policy | Used stronger password `N@utiLus#Str0ng!2026` with more varied characters |

---

## Azure SQL Password Requirements

Azure SQL Server enforces the following password policy:
- Minimum **8 characters**
- Must contain characters from **3 of these 4 categories**:
  - Uppercase letters (A–Z)
  - Lowercase letters (a–z)
  - Numbers (0–9)
  - Special characters (`!`, `@`, `#`, `$`, etc.)
- Must **not contain** the admin username (`devops-admin`)
- Must **not be a common password**

---

## Verification Output

```
Name          Status    Edition    MaxSizeBytes    BackupRedundancy
------------  --------  ---------  --------------  ----------------
devops-sqldb  Online    Basic      2147483648      Local

Name                  FQDN                                              PublicAccess    AdminLogin
--------------------  ------------------------------------------------  --------------  ------------
devops-server-4878    devops-server-4878.database.windows.net           Enabled         devops-admin
```

| Item | Value |
|---|---|
| Database Name | `devops-sqldb` ✅ |
| Server Name | `devops-server-4878` ✅ |
| Region | South Central US ✅ |
| Edition | Basic ✅ |
| Max Size | 2 GiB (2147483648 bytes) ✅ |
| Backup Redundancy | Local ✅ |
| Public Access | Enabled ✅ |
| Status | Online (Ready) ✅ |

---

## Key Concepts

**Azure SQL Architecture — Server vs Database**
Azure SQL has a two-tier structure:
- **SQL Server** — a logical container that holds databases and manages authentication, firewall rules, and regional placement. It is not a physical server.
- **SQL Database** — the actual database instance with its own compute, storage, and performance settings.

**What is the Basic Tier?**
The Basic tier uses DTUs (Database Transaction Units) — a bundled measure of CPU, memory, and I/O. Basic provides 5 DTUs and up to 2 GB storage, making it suitable for small, low-traffic workloads, development, and testing. It is the most cost-effective Azure SQL option.

**Backup Storage Redundancy Options**
| Option | Description |
|---|---|
| `Local` | Copies stored in same datacenter — lowest cost |
| `Zone` | Copies replicated across availability zones |
| `Geo` | Copies replicated to a paired Azure region — highest resilience |

`Local` was chosen as per the task requirement for locally-redundant backup storage.

**Azure SQL Firewall Rules**
Azure SQL Server has its own built-in firewall separate from NSGs. Even if the Azure network allows traffic, the SQL server firewall must explicitly permit the source IP. Setting `0.0.0.0` to `255.255.255.255` opens access from all IPs — appropriate for lab environments but should be restricted to specific IPs in production.

**FQDN for Database Connection**
Once created, the database is accessible via its fully qualified domain name:
```
devops-server-4878.database.windows.net
```
This FQDN is used in connection strings for applications connecting to the database.
