# Task 9 - Attach a Network Interface (NIC) to a Virtual Machine on Azure

## Overview

This task is part of the Nautilus DevOps team's incremental cloud migration strategy to Microsoft Azure. A Network Interface Card (NIC) is the interconnection between a Virtual Machine and the underlying virtual network. This task focuses on attaching an existing NIC to an existing VM, enabling additional network connectivity for the VM.

---

## Objectives

- Attach the existing network interface `nautilus-nic` to the existing virtual machine `nautilus-vm`
- Ensure the NIC's status is **attached** after the operation
- Ensure the VM is in a **running** state after the operation

---

## Prerequisites

- Access to an active Azure subscription or lab credentials
- Azure CLI installed and authenticated
- An existing Resource Group
- An existing VM named `nautilus-vm` in the `westus` region
- An existing NIC named `nautilus-nic`

---

## Tools Used

| Tool | Purpose |
|------|---------|
| Azure CLI | Attaching NIC to VM via command line |
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

### 3. Stop and Deallocate the VM

> **Important:** Azure requires the VM to be in a deallocated (stopped) state before attaching or detaching a NIC. A running VM with a single NIC cannot have additional NICs added while running.

```bash
az vm deallocate \
  --name nautilus-vm \
  --resource-group <ResourceGroupName>
```

Wait for the command to complete before proceeding.

### 4. Attach the NIC to the VM

```bash
az vm nic add \
  --vm-name nautilus-vm \
  --nics nautilus-nic \
  --resource-group <ResourceGroupName>
```

### 5. Start the VM

```bash
az vm start \
  --name nautilus-vm \
  --resource-group <ResourceGroupName>
```

### 6. Verify the NIC is Attached

```bash
az vm nic list \
  --vm-name nautilus-vm \
  --resource-group <ResourceGroupName> \
  --output table
```

### 7. Verify the VM is Running

```bash
az vm show \
  --name nautilus-vm \
  --resource-group <ResourceGroupName> \
  --show-details \
  --query "{Name:name, PowerState:powerState}" \
  -o table
```

---

## Expected Output

**NIC list:**
```
ResourceGroup
----------------------------
kml_rg_main-<id>
```

**VM power state:**
```
Name          PowerState
------------  --------------
nautilus-vm   VM running
```

---

## Key Concepts

- **Network Interface Card (NIC):** A NIC is the component that connects a VM to a Virtual Network. Each VM must have at least one NIC, and multiple NICs can be attached for advanced networking scenarios.
- **VM Deallocation:** Unlike simply stopping a VM, deallocation releases the compute resources. It is required before making structural changes to a VM such as adding or removing NICs or changing VM size.
- **Primary vs Secondary NIC:** The first NIC attached to a VM is the primary NIC. Additional NICs are secondary and can be used for traffic separation, security, or connecting to multiple VNets.
- **NIC and Subnet Association:** A NIC is always associated with a specific subnet within a VNet, defining which network the VM communicates on through that interface.

---

## Notes

- Always deallocate the VM before attaching or detaching NICs — attempting to do so on a running VM will result in an error.
- After starting the VM back up, allow a minute or two for full initialization before verifying.
- The VM size determines the maximum number of NICs that can be attached. For example, `Standard_B1s` supports only 2 NICs.
- Detaching a NIC follows the same process — deallocate, detach, then restart.

---


