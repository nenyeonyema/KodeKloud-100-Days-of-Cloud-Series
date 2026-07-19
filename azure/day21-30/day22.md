# Task 22 — Create Azure VM with Static Public IP (datacenter-vm)

## Overview

The Nautilus DevOps Team received a request to create an Azure Virtual Machine with a Static Public IP address to ensure consistent, reliable access to a hosted application. The VM is named `datacenter-vm` and the static public IP is named `datacenter-pip`.

---

## Requirements

| Parameter | Value |
|---|---|
| VM Name | `datacenter-vm` |
| Public IP Name | `datacenter-pip` |
| Public IP Type | Static |
| VM Size | `Standard_B1s` |
| OS Image | Ubuntu 22.04 LTS |
| Admin Username | `azureuser` |
| SSH Access | Key-based (no password) |
| Region | Central US |

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
az group list --output table
```

### 2. Check for Existing SSH Key, Generate if Missing
```bash
if [ -f ~/.ssh/id_rsa.pub ]; then
    echo "SSH key already exists:"
    cat ~/.ssh/id_rsa.pub
else
    ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa -N ""
    echo "SSH key created:"
    cat ~/.ssh/id_rsa.pub
fi
```

### 3. Create Static Public IP
```bash
az network public-ip create \
    --resource-group kml_rg_main-81a15c42c5544b50 \
    --name datacenter-pip \
    --allocation-method Static \
    --sku Standard \
    --location centralus
```

### 4. Create the VM
```bash
az vm create \
    --resource-group kml_rg_main-81a15c42c5544b50 \
    --name datacenter-vm \
    --image Ubuntu2204 \
    --size Standard_B1s \
    --admin-username azureuser \
    --ssh-key-values ~/.ssh/id_rsa.pub \
    --location centralus \
    --public-ip-address datacenter-pip \
    --public-ip-sku Standard \
    --os-disk-size-gb 30 \
    --storage-sku Standard_LRS \
    --nsg-rule SSH
```

**Flags explained:**
| Flag | Reason |
|---|---|
| `--public-ip-address datacenter-pip` | Associates the pre-created static IP with the VM |
| `--os-disk-size-gb 30` | Keeps disk within lab policy (≤128GB, no Premium SKU) |
| `--storage-sku Standard_LRS` | Required by lab policy — no Premium storage allowed |
| `--nsg-rule SSH` | Explicitly opens port 22 at VM creation time |

### 5. Wait for VM to be Fully Running
```bash
az vm wait \
    --resource-group kml_rg_main-81a15c42c5544b50 \
    --name datacenter-vm \
    --created

az vm get-instance-view \
    --resource-group kml_rg_main-81a15c42c5544b50 \
    --name datacenter-vm \
    --query "instanceView.statuses[*].displayStatus" \
    -o table
```

### 6. Verify Static IP
```bash
az network public-ip show \
    --resource-group kml_rg_main-81a15c42c5544b50 \
    --name datacenter-pip \
    --query "{IPAddress:ipAddress, AllocationMethod:publicIPAllocationMethod, SKU:sku.name}" \
    -o table
```

### 7. Test SSH Access
```bash
PUBLIC_IP=$(az network public-ip show \
    --resource-group kml_rg_main-81a15c42c5544b50 \
    --name datacenter-pip \
    --query ipAddress -o tsv)

echo "Connecting to: $PUBLIC_IP"

ssh -i ~/.ssh/id_rsa \
    -o StrictHostKeyChecking=no \
    -o ConnectTimeout=15 \
    azureuser@$PUBLIC_IP \
    "echo 'SSH SUCCESS - hostname: $(hostname)'"
```

---

## Issues Encountered & Fixes

| Issue | Cause | Fix |
|---|---|---|
| `AuthorizationFailed` mid-task | Azure lab session expired quickly | Re-ran `az login` to refresh credentials before continuing |
| SSH access failed on previous task (devops-vm) | NSG port 22 not explicitly opened | Added `--nsg-rule SSH` at VM creation time |
| Wrong resource group used in earlier attempt | RG name assumed instead of confirmed | Always run `az group list` first and use exact RG name |

---

## Verification Output

```
Connecting to: 20.9.19.232
SSH SUCCESS - hostname: azure-client
```

| Item | Value |
|---|---|
| VM Name | `datacenter-vm` |
| Public IP Name | `datacenter-pip` |
| Public IP Address | `20.9.19.232` |
| IP Type | Static |
| Region | Central US |
| SSH | ✅ Successful |

---

## Key Concepts

**Why use `--nsg-rule SSH` at creation?**
By default, `az vm create` may not always create an NSG rule for SSH depending on the environment. Explicitly passing `--nsg-rule SSH` guarantees port 22 is open in the Network Security Group at creation time, preventing SSH timeouts after deployment.

**Why `az vm wait --created`?**
Azure VM provisioning is asynchronous. The `az vm create` command may return before the VM is fully ready to accept connections. Using `az vm wait --created` ensures the VM reaches the `Succeeded` provisioning state before attempting SSH.

**Session Expiry in Lab Environments**
Azure lab credentials expire quickly (within the 1-hour lab window). If you get `AuthorizationFailed` mid-task, simply re-run `az login` with the same credentials to refresh the token and continue.
