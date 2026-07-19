# Task 23 — Create Azure VM with Nginx Web Server via User Data

## Overview

The Nautilus DevOps Team needed to deploy a VM that automatically installs and starts Nginx upon launch, making it immediately accessible as a web server on port 80 from the internet. The VM is named `xfusion-vm` and is configured using a custom cloud-init script passed via `--custom-data`.

---

## Requirements

| Parameter | Value |
|---|---|
| VM Name | `xfusion-vm` |
| OS Image | Ubuntu 22.04 LTS |
| VM Size | `Standard_B1s` |
| Admin Username | `azureuser` |
| SSH Access | Key-based (no password) |
| Custom Script | Install Nginx, start and enable the service |
| Port 80 | Open to internet (HTTP) |
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
az group list --output table
```

### 2. Generate SSH Key if Missing
```bash
[ -f ~/.ssh/id_rsa.pub ] || ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa -N ""
```

### 3. Create the Cloud-Init Script
```bash
cat > /tmp/cloud-init.txt << 'EOF'
#!/bin/bash
apt-get update -y
apt-get install -y nginx
systemctl start nginx
systemctl enable nginx
EOF
```

> This script is passed to the VM at launch and executed automatically by cloud-init during first boot.

### 4. Create the VM with User Data
```bash
az vm create \
    --resource-group kml_rg_main-81a15c42c5544b50 \
    --name xfusion-vm \
    --image Ubuntu2204 \
    --size Standard_B1s \
    --admin-username azureuser \
    --ssh-key-values ~/.ssh/id_rsa.pub \
    --location eastus \
    --os-disk-size-gb 30 \
    --storage-sku Standard_LRS \
    --nsg-rule SSH \
    --custom-data /tmp/cloud-init.txt
```

**Flags explained:**
| Flag | Reason |
|---|---|
| `--custom-data /tmp/cloud-init.txt` | Passes the cloud-init script to run on first boot |
| `--nsg-rule SSH` | Opens port 22 at creation time |
| `--os-disk-size-gb 30` | Keeps disk within lab policy |
| `--storage-sku Standard_LRS` | Required by lab policy |

### 5. Open Port 80 for HTTP Traffic
```bash
az vm open-port \
    --resource-group kml_rg_main-81a15c42c5544b50 \
    --name xfusion-vm \
    --port 80 \
    --priority 100
```

### 6. Wait for VM to be Ready
```bash
az vm wait \
    --resource-group kml_rg_main-81a15c42c5544b50 \
    --name xfusion-vm \
    --created

az vm get-instance-view \
    --resource-group kml_rg_main-81a15c42c5544b50 \
    --name xfusion-vm \
    --query "instanceView.statuses[*].displayStatus" \
    -o table
```

### 7. Get Public IP
```bash
PUBLIC_IP=$(az vm show \
    --resource-group kml_rg_main-81a15c42c5544b50 \
    --name xfusion-vm \
    --show-details \
    --query publicIps -o tsv)

echo "VM Public IP: $PUBLIC_IP"
```

### 8. Wait for Cloud-Init to Finish then Verify Nginx
```bash
# Wait for cloud-init to complete Nginx installation
sleep 60

# Test HTTP response
curl -s -o /dev/null -w "HTTP Status: %{http_code}\n" http://$PUBLIC_IP

# Check Nginx status via SSH
ssh -i ~/.ssh/id_rsa \
    -o StrictHostKeyChecking=no \
    azureuser@$PUBLIC_IP \
    "systemctl status nginx --no-pager | head -5"
```

---

## Issues Encountered & Fixes

| Issue | Cause | Fix |
|---|---|---|
| Port 80 not open after VM creation | `--nsg-rule SSH` only opens port 22 | Added `az vm open-port --port 80` separately after VM creation |
| Nginx not ready immediately | Cloud-init runs asynchronously after VM boots | Added `sleep 60` to allow cloud-init to complete before testing |

---

## Verification Output

[O```
HTTP Status: 200

● nginx.service - A high performance web server
   Active: active (running)
```

| Item | Value |
|---|---|
| VM Name | `xfusion-vm` |
| Region | East US |
| Nginx | ✅ Installed, enabled, running |
| Port 80 | ✅ Open and serving HTTP 200 |
| SSH | ✅ Accessible |

---

## Key Concepts

**What is Cloud-Init / User Data?**
Cloud-init is a standard tool used to customize cloud VMs during their first boot. By passing a shell script via `--custom-data`, Azure executes it automatically when the VM starts for the first time — no manual SSH required. This is the standard way to bootstrap VMs with software at launch time.

**Why `sleep 60` before testing?**
The VM reaches `running` state before cloud-init finishes executing. The `az vm wait --created` confirms the VM is provisioned, but Nginx installation takes additional time. Waiting 60 seconds ensures the install script has completed before we test HTTP.

**Why open port 80 separately?**
The `--nsg-rule` flag at creation only supports `SSH` or `RDP` as shorthand values. HTTP (port 80) must be opened with a separate `az vm open-port` command after the VM is created.
