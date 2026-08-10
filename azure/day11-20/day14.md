# Task 14 - Create a Managed Disk on Azure

## Overview

This task is part of the Nautilus DevOps team's incremental cloud migration strategy to Microsoft Azure. Managed disks are Azure-managed block-level storage volumes used with Azure Virtual Machines. This task focuses on creating a managed disk with specific requirements including disk name, type, and size as part of the team's gradual infrastructure migration.

---

## Objectives

- Create a managed disk named `datacenter-disk`
- Set the disk type to `Standard_LRS`
- Set the disk size to `2 GiB`

---

## Prerequisites

- Access to an active Azure subscription or lab credentials
- Azure CLI installed and authenticated
- An existing Resource Group

---

## Tools Used

| Tool | Purpose |
|------|---------|
| Azure CLI | Creating the managed disk via command line |
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

Note the resource group name from the output.

### 3. Create the Managed Disk

```bash
az disk create \
  --name datacenter-disk \
  --resource-group <ResourceGroupName> \
  --sku Standard_LRS \
  --size-gb 2
```

### 4. Verify the Managed Disk was Created

```bash
az disk show \
  --name datacenter-disk \
  --resource-group <ResourceGroupName> \
  --query "{Name:name, SKU:sku.name, SizeGB:diskSizeGb, ProvisioningState:provisioningState}" \
  -o table
```

---

## Expected Output

```
Name              SKU           SizeGB    ProvisioningState
----------------  ------------  --------  -------------------
datacenter-disk   Standard_LRS  2         Succeeded
```

---

## Managed Disk Types Comparison

| SKU | Full Name | Use Case | Cost |
|-----|-----------|----------|------|
| `Standard_LRS` | Standard HDD Locally Redundant Storage | Dev/test, backups, infrequent access | Lowest |
| `StandardSSD_LRS` | Standard SSD Locally Redundant Storage | Web servers, lightly used apps | Low |
| `Premium_LRS` | Premium SSD Locally Redundant Storage | Production, I/O intensive workloads | Medium |
| `UltraSSD_LRS` | Ultra Disk Locally Redundant Storage | Mission critical, high throughput | Highest |

---

## Key Concepts

- **Managed Disk:** A virtualized disk in Azure that is fully managed by the platform. Unlike unmanaged disks, you don't need to manage storage accounts — Azure handles availability, redundancy, and scaling automatically.
- **Standard_LRS (Locally Redundant Storage):** Replicates data three times within a single data center. It is the most cost-effective option and suitable for dev/test environments and workloads tolerant of lower IOPS.
- **Disk Size:** Azure managed disks come in predefined sizes. The minimum size is 1 GiB and sizes can be increased but not decreased after creation.
- **OS Disk vs Data Disk:** VMs have an OS disk (contains the operating system) and optionally one or more data disks. Managed disks can serve as either.
- **LRS (Locally Redundant Storage):** Data is replicated synchronously three times within a single physical location in the primary region, protecting against server rack and drive failures.

---

## Notes

- Managed disks can be attached to a VM after creation using:
  ```bash
  az vm disk attach \
    --vm-name <VMName> \
    --resource-group <ResourceGroupName> \
    --name datacenter-disk
  ```
- Disk size can only be **increased**, never decreased after creation.
- A managed disk not attached to any VM still incurs storage costs.
- To list all managed disks in a resource group:
  ```bash
  az disk list \
    --resource-group <ResourceGroupName> \
    --output table
  ```
- Snapshots of managed disks can be created for backup purposes using `az snapshot create`.

---

