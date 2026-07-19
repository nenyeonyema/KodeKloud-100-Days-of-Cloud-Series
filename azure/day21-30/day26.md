# Task 26 — Create Public VNet, Subnet and VM with Internet Access

## Overview

The Nautilus DevOps Team needed to set up a public-facing Virtual Network (VNet) with a subnet that auto-assigns public IPs to resources. A VM named `nautilus-pub-vm` was deployed within this VNet with SSH access open from the internet. This setup enables the Networking Team to deploy and manage public-facing applications.

---

## Requirements

| Parameter | Value |
|---|---|
| VNet Name | `nautilus-pub-vnet` |
| VNet CIDR | `10.0.0.0/16` |
| Subnet Name | `nautilus-pub-subnet` |
| Subnet CIDR | `10.0.1.0/24` |
| Public IP | Auto-assigned to resources |
| VM Name | `nautilus-pub-vm` |
| VM Size | `Standard_B1s` |
| OS Image | Ubuntu 22.04 LTS |
| Admin Username | `azureuser` |
| SSH Port 22 | Open from internet |
| NSG Name | `nautilus-pub-nsg` |
| Region | East US |

---

## Prerequisites

- Azure CLI installed and configured on the azure-client host
- Access to Azure credentials via `showcreds` command
- SSH key pair available on azure-client

---

## Steps Performed

### 1. Login to Azure
```bash
az login -u "<username>" -p "<password>"
az group list --output table
```

### 2. Create the VNet
```bash
az network vnet create \
    --resource-group kml_rg_main-6264039ba91f41b8 \
    --name nautilus-pub-vnet \
    --location eastus \
    --address-prefix 10.0.0.0/16
```

### 3. Create the Public Subnet
```bash
az network vnet subnet create \
    --resource-group kml_rg_main-6264039ba91f41b8 \
    --vnet-name nautilus-pub-vnet \
    --name nautilus-pub-subnet \
    --address-prefix 10.0.1.0/24
```

### 4. Generate SSH Key if Missing
```bash
[ -f ~/.ssh/id_rsa.pub ] || ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa -N ""
```

### 5. Create NSG with SSH Rule
```bash
az network nsg create \
    --resource-group kml_rg_main-6264039ba91f41b8 \
    --name nautilus-pub-nsg \
    --location eastus

az network nsg rule create \
    --resource-group kml_rg_main-6264039ba91f41b8 \
    --nsg-name nautilus-pub-nsg \
    --name allow-ssh \
    --protocol Tcp \
    --direction Inbound \
    --priority 100 \
    --source-address-prefix Internet \
    --source-port-range '*' \
    --destination-address-prefix '*' \
    --destination-port-range 22 \
    --access Allow
```

### 6. Create Public IP (Standard SKU)
```bash
az network public-ip create \
    --resource-group kml_rg_main-6264039ba91f41b8 \
    --name nautilus-pub-vm-pip \
    --location eastus \
    --allocation-method Static \
    --sku Standard
```

> **Note:** Basic SKU was initially attempted but failed — the lab subscription had reached the Basic SKU limit of 0. Standard SKU with Static allocation was used instead.

### 7. Create NIC Tied to Subnet + NSG + Public IP
```bash
az network nic create \
    --resource-group kml_rg_main-6264039ba91f41b8 \
    --name nautilus-pub-vm-nic \
    --location eastus \
    --vnet-name nautilus-pub-vnet \
    --subnet nautilus-pub-subnet \
    --network-security-group nautilus-pub-nsg \
    --public-ip-address nautilus-pub-vm-pip
```

### 8. Create the VM Using the Pre-configured NIC
```bash
az vm create \
    --resource-group kml_rg_main-6264039ba91f41b8 \
    --name nautilus-pub-vm \
    --location eastus \
    --image Ubuntu2204 \
    --size Standard_B1s \
    --admin-username azureuser \
    --ssh-key-values ~/.ssh/id_rsa.pub \
    --nics nautilus-pub-vm-nic \
    --os-disk-size-gb 30 \
    --storage-sku Standard_LRS
```

### 9. Wait for VM to be Ready
```bash
az vm wait \
    --resource-group kml_rg_main-6264039ba91f41b8 \
    --name nautilus-pub-vm \
    --created

az vm get-instance-view \
    --resource-group kml_rg_main-6264039ba91f41b8 \
    --name nautilus-pub-vm \
    --query "instanceView.statuses[*].displayStatus" \
    -o table
```

### 10. Verify and Test SSH
```bash
PUBLIC_IP=$(az vm show \
    --resource-group kml_rg_main-6264039ba91f41b8 \
    --name nautilus-pub-vm \
    --show-details \
    --query publicIps -o tsv)

echo "Public IP: $PUBLIC_IP"

az network vnet subnet show \
    --resource-group kml_rg_main-6264039ba91f41b8 \
    --vnet-name nautilus-pub-vnet \
    --name nautilus-pub-subnet \
    --query "{Name:name, Prefix:addressPrefix, NSG:networkSecurityGroup.id}" \
    -o table

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
| `IPv4BasicSkuPublicIpCountLimitReached` | Lab subscription reached limit of 0 Basic SKU public IPs | Switched to `--sku Standard --allocation-method Static` |

---

## Verification Output

```
Name                  Prefix        NSG
--------------------  ------------  -------------------------------------------
nautilus-pub-subnet   10.0.1.0/24   .../nautilus-pub-nsg

SSH SUCCESS - hostname: nautilus-pub-vm
```

| Item | Value |
|---|---|
| VNet | `nautilus-pub-vnet` — `10.0.0.0/16` ✅ |
| Subnet | `nautilus-pub-subnet` — `10.0.1.0/24` ✅ |
| NSG | `nautilus-pub-nsg` — SSH port 22 open ✅ |
| Public IP | Auto-assigned via NIC ✅ |
| VM | `nautilus-pub-vm` running ✅ |
| SSH | ✅ Successful |

---

## Key Concepts

**Why Create NIC Explicitly?**
When deploying a VM into a specific VNet/subnet with a pre-created NSG and public IP, creating the NIC separately gives precise control over all network settings before the VM is created. Using `--nics` in `az vm create` then attaches this fully configured NIC directly.

**What Makes a Subnet "Public"?**
In Azure, a subnet is considered public when resources within it have public IPs assigned and the NSG allows inbound traffic from the internet. There is no separate "public subnet" toggle like in AWS — internet accessibility is controlled entirely by NSG rules and public IP assignment.

**Why Standard SKU over Basic SKU for Public IP?**
Basic SKU public IPs are being deprecated by Microsoft. Standard SKU IPs are zone-redundant, more secure (closed by default), and required for use with Standard SKU load balancers. Lab subscriptions may also cap Basic SKU IP counts at 0, making Standard the only viable option.

**Source `Internet` in NSG Rule**
Using `Internet` as the source address prefix in an NSG rule means "any traffic coming from outside Azure's virtual network space" — effectively all public internet traffic. This is the correct value for allowing public SSH access from anywhere.
