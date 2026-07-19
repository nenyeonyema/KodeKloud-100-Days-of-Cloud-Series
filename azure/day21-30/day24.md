# Task 24 — Create Azure VM with Password-less SSH Access

## Overview

The Nautilus DevOps Team needed to set up a new Virtual Machine on Azure that can be accessed securely from the azure-client host using password-less SSH. The VM is named `nautilus-vm` and is deployed in the West US region. An SSH key generated on the azure-client host is used to enable secure, password-less access.

---

## Requirements

| Parameter | Value |
|---|---|
| VM Name | `nautilus-vm` |
| OS Image | Ubuntu 22.04 LTS |
| VM Size | `Standard_B1s` |
| Admin Username | `azureuser` |
| SSH Access | Password-less, key-based only |
| SSH Key | Generated on azure-client host |
| Region | West US |

---

## Prerequisites

- Azure CLI installed and configured on the azure-client host
- Access to Azure credentials via `showcreds` command
- SSH key pair to be checked/generated on azure-client

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

> The task requires checking if a key already exists before creating a new one. The `-N ""` flag sets an empty passphrase, enabling fully password-less authentication.

### 3. Create the VM
```bash
az vm create \
    --resource-group kml_rg_main-a33ac06c85fc4e0b \
    --name nautilus-vm \
    --image Ubuntu2204 \
    --size Standard_B1s \
    --admin-username azureuser \
    --ssh-key-values ~/.ssh/id_rsa.pub \
    --location westus \
    --os-disk-size-gb 30 \
    --storage-sku Standard_LRS \
    --nsg-rule SSH
```

**Flags explained:**
| Flag | Reason |
|---|---|
| `--ssh-key-values ~/.ssh/id_rsa.pub` | Injects the public key into the VM's `~/.ssh/authorized_keys` |
| `--nsg-rule SSH` | Opens port 22 in the NSG at creation time |
| `--os-disk-size-gb 30` | Keeps disk within lab policy |
| `--storage-sku Standard_LRS` | Required by lab policy |

### 4. Wait for VM to be Fully Running
```bash
az vm wait \
    --resource-group kml_rg_main-a33ac06c85fc4e0b \
    --name nautilus-vm \
    --created

az vm get-instance-view \
    --resource-group kml_rg_main-a33ac06c85fc4e0b \
    --name nautilus-vm \
    --query "instanceView.statuses[*].displayStatus" \
    -o table
```

### 5. Get Public IP and Test SSH
```bash
PUBLIC_IP=$(az vm show \
    --resource-group kml_rg_main-a33ac06c85fc4e0b \
    --name nautilus-vm \
    --show-details \
    --query publicIps -o tsv)

echo "VM Public IP: $PUBLIC_IP"

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
| None | Task completed cleanly using lessons from previous tasks | Used `--nsg-rule SSH`, correct RG, and `Standard_LRS` from the start |

---

## Verification Output

```
VM Public IP: 40.75.137.167
SSH SUCCESS - hostname: azure-client
```

| Item | Value |
|---|---|
| VM Name | `nautilus-vm` |
| Public IP | `40.75.137.167` |
| Region | West US |
| VM Size | `Standard_B1s` |
| SSH User | `azureuser` |
| SSH Key | `~/.ssh/id_rsa` |
| Password-less SSH | ✅ Confirmed |

---

## Key Concepts

**How Password-less SSH Works**
When you run `ssh-keygen`, it creates a key pair:
- **Private key** (`~/.ssh/id_rsa`) — stays on the azure-client host, never shared
- **Public key** (`~/.ssh/id_rsa.pub`) — injected into the VM via `--ssh-key-values`

Azure places the public key in `/home/azureuser/.ssh/authorized_keys` on the VM. When you SSH in, your private key is matched against the stored public key — no password needed.

**Why Check for Existing Key First?**
If a key already exists and you generate a new one, the old key becomes invalid for any VM already configured with it. The check-first approach preserves existing key-based access to other VMs.

**Why `-N ""` (Empty Passphrase)?**
Setting an empty passphrase means the private key itself is not encrypted. This enables fully automated, password-less SSH — no interactive prompt required. In production environments, a passphrase would be set for additional security, but for lab automation this is the standard approach.

**Why `--nsg-rule SSH` at Creation?**
Without this flag, the NSG may not have port 22 open, causing SSH connection timeouts. Always include `--nsg-rule SSH` when creating VMs that need SSH access.
