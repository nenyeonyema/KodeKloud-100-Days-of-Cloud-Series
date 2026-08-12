# Task 3 - Create an Azure Virtual Machine (VM) via the Azure CLI

## Overview

This task is part of the Nautilus DevOps team's incremental cloud migration strategy to Microsoft Azure. This task mirrors Task 2 but uses the Azure CLI instead of the Azure Portal, demonstrating the power of infrastructure provisioning via the command line. Creating VMs through the CLI is faster, repeatable, and scriptable — making it the preferred approach in DevOps workflows.

---

## Objectives

- Create a Virtual Machine named `devops-vm` in the `eastus` region
- Use the existing Resource Group
- Use the **Ubuntu 24.04 LTS** image
- Set the VM size to `Standard_B1s`
- Attach a default Network Security Group (NSG) that allows inbound SSH (port 22)
- Attach a **30 GB Standard HDD** storage disk
- Verify SSH access to the VM after creation

---

## Prerequisites

- Access to an active Azure subscription or lab credentials
- Azure CLI installed and authenticated
- An existing Resource Group

---

## Tools Used

| Tool | Purpose |
|------|---------|
| Azure CLI | Creating the VM via the command line |
| SSH | Verifying connectivity to the VM |

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

### 3. Create the Virtual Machine

```bash
az vm create \
  --name devops-vm \
  --resource-group <ResourceGroupName> \
  --location eastus \
  --image Ubuntu2404 \
  --size Standard_B1s \
  --os-disk-size-gb 30 \
  --storage-sku Standard_LRS \
  --admin-username azureuser \
  --generate-ssh-keys \
  --public-ip-sku Standard \
  --nsg-rule SSH
```

> `--generate-ssh-keys` automatically creates an SSH key pair at `~/.ssh/id_rsa` and `~/.ssh/id_rsa.pub` on the client host if one does not already exist.

### 4. Verify the VM was Created

```bash
az vm show \
  --name devops-vm \
  --resource-group <ResourceGroupName> \
  --show-details \
  --query "{Name:name, State:powerState, Size:hardwareProfile.vmSize, PublicIP:publicIps}" \
  -o table
```

### 5. Get the VM's Public IP Address

```bash
az vm show \
  --name devops-vm \
  --resource-group <ResourceGroupName> \
  --show-details \
  --query "publicIps" \
  -o tsv
```

### 6. Open Port 22 if Not Already Open

```bash
az vm open-port \
  --name devops-vm \
  --resource-group <ResourceGroupName> \
  --port 22
```

### 7. SSH into the VM

```bash
ssh azureuser@<PublicIPFromStep5>
```

---

## Expected Output

**VM creation:**
```json
{
  "fqdns": "",
  "id": "/subscriptions/.../resourceGroups/.../providers/Microsoft.Compute/virtualMachines/devops-vm",
  "location": "eastus",
  "macAddress": "xx-xx-xx-xx-xx-xx",
  "powerState": "VM running",
  "privateIpAddress": "10.0.0.4",
  "publicIpAddress": "<PublicIP>",
  "resourceGroup": "<ResourceGroupName>",
  "zones": ""
}
```

**Successful SSH login:**
```
Welcome to Ubuntu 24.04 LTS (GNU/Linux 6.x.x-azure x86_64)
azureuser@devops-vm:~$
```

---

## VM Configuration Summary

| Property | Value |
|----------|-------|
| Name | `devops-vm` |
| Region | `East US` |
| Image | Ubuntu Server 24.04 LTS |
| Size | Standard_B1s (1 vCPU, 1 GiB RAM) |
| OS Disk Type | Standard HDD (Standard_LRS) |
| OS Disk Size | 30 GiB |
| Inbound Ports | SSH (22) |
| Admin User | `azureuser` |
| Authentication | SSH key pair (auto-generated) |

---

## CLI vs Portal Comparison

| Step | Azure Portal | Azure CLI |
|------|-------------|-----------|
| Authentication | Browser login | `az login` |
| VM Creation | Fill multiple form tabs | Single `az vm create` command |
| Speed | Slower (manual input) | Faster (one command) |
| Repeatability | Manual each time | Scriptable and automatable |
| SSH Key Handling | Download `.pem` manually | Auto-generated at `~/.ssh/` |
| Best For | Learning, one-off tasks | DevOps workflows, automation |

---

## Key Concepts

- **`az vm create`:** The primary Azure CLI command for provisioning a virtual machine. It creates the VM along with associated resources (NIC, public IP, NSG, OS disk) in a single command.
- **`--generate-ssh-keys`:** Automatically generates an RSA key pair if one does not exist at `~/.ssh/id_rsa`. The public key is installed on the VM at creation, enabling immediate passwordless SSH access.
- **`--storage-sku Standard_LRS`:** Sets the OS disk type to Standard HDD with Locally Redundant Storage, the most cost-effective disk option.
- **`--nsg-rule SSH`:** Creates a Network Security Group with a default inbound rule allowing SSH traffic on port 22.
- **`--os-disk-size-gb 30`:** Sets the OS disk size to 30 GiB at creation time.
- **Ubuntu2404:** The Azure CLI image alias for Ubuntu Server 24.04 LTS. Use `az vm image list --output table` to find other available image aliases.

---

## Notes

- The `--generate-ssh-keys` flag stores the private key at `~/.ssh/id_rsa` on the client host. Keep this file secure and never share it.
- To list all available Ubuntu images on Azure:
  ```bash
  az vm image list \
    --publisher Canonical \
    --output table \
    --all
  ```
- To list available VM sizes in the `eastus` region:
  ```bash
  az vm list-sizes \
    --location eastus \
    --output table
  ```
- The `az vm create` command also accepts `--authentication-type password` and `--admin-password` if SSH key authentication is not preferred.
- Always deallocate VMs when not in use in lab environments:
  ```bash
  az vm deallocate \
    --name devops-vm \
    --resource-group <ResourceGroupName>
  ```

---

