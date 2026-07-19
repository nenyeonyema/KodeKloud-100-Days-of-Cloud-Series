# Task 21 — Create Azure VM with Static Public IP

## Overview

The Nautilus DevOps Team received a request to create an Azure Virtual Machine with a Static Public IP address to ensure consistent, reliable access to a hosted application. The VM is named `xfusion-vm` and the static public IP is named `xfusion-pip`.

---

## Requirements

| Parameter | Value |
|---|---|
| VM Name | `xfusion-vm` |
| Public IP Name | `xfusion-pip` |
| Public IP Type | Static |
| VM Size | `Standard_B1s` |
| OS Image | Ubuntu 22.04 LTS |
| Admin Username | `azureuser` |
| SSH Access | Key-based (no password) |
| Region | East US |

---

## Prerequisites

- Azure CLI installed and configured on the azure-client host
- Access to Azure credentials via `showcreds` command
- SSH key pair available or to be generated on azure-client

---

## Steps Performed

### 1. Login to Azure
```bash
az login -u "<username>" -p "<password>"
```

### 2. Check Existing Resource Group
```bash
az group list --output table
```
> The existing resource group `kml_rg_main-505a37762a57437e` in `eastus` was used instead of creating a new one.

### 3. Generate SSH Key Pair
```bash
ssh-keygen -t rsa -b 4096 -f ~/.ssh/xfusion_rsa -N ""
```
- Private key: `~/.ssh/xfusion_rsa`
- Public key: `~/.ssh/xfusion_rsa.pub`

### 4. Create Static Public IP
```bash
az network public-ip create \
    --resource-group kml_rg_main-505a37762a57437e \
    --name xfusion-pip \
    --allocation-method Static \
    --sku Standard \
    --location eastus
```
> `--allocation-method Static` ensures the IP address never changes, even after VM restarts.

### 5. Create the VM
```bash
az vm create \
    --resource-group kml_rg_main-505a37762a57437e \
    --name xfusion-vm \
    --image Ubuntu2204 \
    --size Standard_B1s \
    --admin-username azureuser \
    --ssh-key-values ~/.ssh/xfusion_rsa.pub \
    --location eastus \
    --public-ip-address xfusion-pip \
    --public-ip-sku Standard \
    --os-disk-size-gb 30 \
    --storage-sku Standard_LRS
```

**Flags explained:**
| Flag | Reason |
|---|---|
| `--public-ip-address xfusion-pip` | Associates the pre-created static IP with the VM |
| `--os-disk-size-gb 30` | Keeps disk within lab policy (≤128GB, no Premium SKU) |
| `--storage-sku Standard_LRS` | Required by lab policy — no Premium storage allowed |

### 6. Open SSH Port
```bash
az vm open-port \
    --resource-group kml_rg_main-505a37762a57437e \
    --name xfusion-vm \
    --port 22
```

### 7. Verify Static IP
```bash
az network public-ip show \
    --resource-group kml_rg_main-505a37762a57437e \
    --name xfusion-pip \
    --query "{IPAddress:ipAddress, AllocationMethod:publicIPAllocationMethod}" \
    -o table
```

### 8. Test SSH Access
```bash
PUBLIC_IP=$(az network public-ip show \
    --resource-group kml_rg_main-505a37762a57437e \
    --name xfusion-pip \
    --query ipAddress -o tsv)

ssh -i ~/.ssh/xfusion_rsa \
    -o StrictHostKeyChecking=no \
    azureuser@$PUBLIC_IP \
    "echo 'SSH SUCCESS - hostname: $(hostname)'"
```

---

## Issues Encountered & Fixes

| Issue | Cause | Fix |
|---|---|---|
| `AuthorizationFailed` on public IP create | Wrong resource group (`nautilus-rg` doesn't exist) | Used existing RG `kml_rg_main-505a37762a57437e` |
| `RequestDisallowedByPolicy` on VM create | Default OS disk uses Premium SKU >128GB | Added `--os-disk-size-gb 30 --storage-sku Standard_LRS` |
| `OperationNotAllowed` on second VM create attempt | Leftover disk from failed deployment conflicting | Deleted failed VM and orphaned disk before retrying |

---

## Verification

```
VM Name:           xfusion-vm
Public IP Name:    xfusion-pip
IP Type:           Static
SSH:               Successful
```

---

## Key Concepts

**Why Static IP?**
A Dynamic IP changes every time the VM is stopped and restarted. A Static IP stays the same permanently, which is essential for applications that need a consistent, bookmarkable endpoint (DNS records, firewall whitelists, client configurations).

**Why `--public-ip-sku Standard`?**
Standard SKU public IPs are zone-redundant and required when pairing with Standard SKU load balancers or when the VM needs predictable, highly available connectivity. Basic SKU is being deprecated.
