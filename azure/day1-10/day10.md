# Task 10 - Attach a Public IP Address to a VM's Network Interface on Azure

## Overview

This task is part of the Nautilus DevOps team's incremental cloud migration strategy to Microsoft Azure. Having already set up a virtual machine and allocated a public IP address, this task focuses on associating the public IP address with the VM's Network Interface Card (NIC), enabling the VM to be reachable from the internet.

---

## Objectives

- Attach the existing public IP address `xfusion-pip` to the network interface of the existing VM `xfusion-vm-pip`
- Ensure the VM is properly assigned the public IP after the operation

---

## Prerequisites

- Access to an active Azure subscription or lab credentials
- Azure CLI installed and authenticated
- An existing Resource Group
- An existing VM named `xfusion-vm-pip`
- An existing Public IP address named `xfusion-pip`

---

## Tools Used

| Tool | Purpose |
|------|---------|
| Azure CLI | Attaching Public IP to NIC via command line |
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

### 3. Get the NIC Name Attached to the VM

```bash
az vm show \
  --name xfusion-vm-pip \
  --resource-group <ResourceGroupName> \
  --query "networkProfile.networkInterfaces[0].id" \
  -o tsv
```

Note the NIC name at the end of the returned resource ID (e.g. `xfusion-vm-pipVMNic`).

### 4. Get the NIC's IP Configuration Name

```bash
az network nic ip-config list \
  --nic-name <NICNameFromStep3> \
  --resource-group <ResourceGroupName> \
  --output table
```

Note the IP config name from the output (e.g. `ipconfigxfusion-vm-pip`).

### 5. List Public IP Details to Get the Full Resource ID

```bash
az network public-ip list --output table
```

Note the full subscription path for use in the next step.

### 6. Attach the Public IP to the NIC

```bash
az network nic ip-config update \
  --nic-name <NICNameFromStep3> \
  --name <IPConfigNameFromStep4> \
  --resource-group <ResourceGroupName> \
  --public-ip-address /subscriptions/<SubscriptionID>/resourceGroups/<ResourceGroupName>/providers/Microsoft.Network/publicIPAddresses/xfusion-pip
```

### 7. Verify the Public IP is Attached

```bash
az vm show \
  --name xfusion-vm-pip \
  --resource-group <ResourceGroupName> \
  --show-details \
  --query "publicIps" \
  -o tsv
```

---

## Expected Output

**IP config list (Step 4):**
```
Name                    Primary    PrivateIPAddress    PrivateIPAddressVersion    PrivateIPAllocationMethod    ProvisioningState
----------------------  ---------  ------------------  -------------------------  ---------------------------  -------------------
ipconfigxfusion-vm-pip  True       10.0.0.4            IPv4                       Dynamic                      Succeeded
```

**Public IP verification (Step 7):**
```
130.131.252.38
```

---

## Key Concepts

- **IP Configuration (ipconfig):** Each NIC has one or more IP configurations. An IP config defines the private IP address and optionally an associated public IP address for the NIC.
- **Public IP Association:** A public IP is not directly attached to a VM — it is attached to the NIC's IP configuration, which is then associated with the VM.
- **Resource ID:** Azure resources can be referenced by their full resource ID path in the format `/subscriptions/<id>/resourceGroups/<rg>/providers/<provider>/<resource>`. This is useful when resources span different resource groups or when CLI shorthand fails.
- **NIC IP Config Naming:** Azure auto-generates IP config names based on the VM or NIC name. Always verify the exact name using `az network nic ip-config list` rather than assuming it is `ipconfig1`.

---

## Notes

- Unlike attaching a NIC, attaching a Public IP to an existing NIC does **not** require the VM to be deallocated first.
- Always use `az network nic ip-config list` to confirm the exact IP config name — it varies per VM and is not always `ipconfig1`.
- If the `--public-ip-address` shorthand fails, use the full resource ID as shown in Step 6.
- The public IP and NIC must be in the **same region** for the association to work.

---


