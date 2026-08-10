# Task 11 - Resize a Virtual Machine on Azure

## Overview

This task is part of the Nautilus DevOps team's incremental cloud migration strategy to Microsoft Azure. During the migration, the team identified an underutilized virtual machine and decided to resize it to better optimize resource usage and cost. This task focuses on changing the VM size from `Standard_B1s` to `Standard_B2s` and ensuring the VM is back in a running state after the resize.

---

## Objectives

- Change the VM size from `Standard_B1s` to `Standard_B2s` for the virtual machine named `devops-vm`
- Ensure the VM is in a **running** state after the size change is complete

---

## Prerequisites

- Access to an active Azure subscription or lab credentials
- Azure CLI installed and authenticated
- An existing Resource Group
- An existing VM named `devops-vm`

---

## Tools Used

| Tool | Purpose |
|------|---------|
| Azure CLI | Resizing the VM via command line |
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

### 3. Verify the Current VM Size

```bash
az vm show \
  --name devops-vm \
  --resource-group <ResourceGroupName> \
  --query "hardwareProfile.vmSize" \
  -o tsv
```

Confirm it shows `Standard_B1s` before proceeding.

### 4. Stop and Deallocate the VM

> **Important:** Azure requires the VM to be deallocated before resizing in most cases, especially when the new size is in a different hardware cluster.

```bash
az vm deallocate \
  --name devops-vm \
  --resource-group <ResourceGroupName>
```

Wait for the command to complete before proceeding.

### 5. Resize the VM

```bash
az vm resize \
  --name devops-vm \
  --resource-group <ResourceGroupName> \
  --size Standard_B2s
```

### 6. Start the VM

```bash
az vm start \
  --name devops-vm \
  --resource-group <ResourceGroupName>
```

### 7. Verify the VM Size and Running State

```bash
az vm show \
  --name devops-vm \
  --resource-group <ResourceGroupName> \
  --show-details \
  --query "{Size:hardwareProfile.vmSize, State:powerState}" \
  -o table
```

---

## Expected Output

**Before resize:**
```
Standard_B1s
```

**After resize and start:**
```
Size            State
--------------  -----------
Standard_B2s    VM running
```

---

## VM Size Comparison

| Property | Standard_B1s | Standard_B2s |
|----------|-------------|-------------|
| vCPUs | 1 | 2 |
| RAM | 1 GiB | 4 GiB |
| Temp Storage | 4 GiB | 8 GiB |
| Max NICs | 2 | 3 |
| Use Case | Light workloads | Moderate workloads |

---

## Key Concepts

- **VM Size:** Defines the compute capacity of a VM including vCPUs, memory, storage, and network bandwidth. Azure offers several size families optimized for different workloads.
- **B-Series (Burstable):** The B-series VMs are cost-effective for workloads that don't need full CPU performance continuously. They accumulate CPU credits when idle and spend them during bursts of activity.
- **VM Deallocation:** Releases the underlying compute resources of a VM. Required before resizing when the target size belongs to a different hardware cluster than the current size.
- **Right-sizing:** The practice of matching VM size to actual workload requirements to optimize cost and performance — a key principle in cloud cost management.

---

## Notes

- Deallocating a VM releases its dynamic public IP if one is assigned — use a static IP to avoid this.
- After resizing, always verify the new size using `az vm show` before submitting the task.
- The B-series is ideal for dev/test environments and workloads with variable CPU usage patterns.
- To check available VM sizes in a region before resizing, run:
  ```bash
  az vm list-vm-resize-options \
    --name devops-vm \
    --resource-group <ResourceGroupName> \
    --output table
  ```

---


