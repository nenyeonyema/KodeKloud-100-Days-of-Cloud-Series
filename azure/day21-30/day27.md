# Task 27 — Create Private VNet, Subnet, NSG and VM

## Overview

The Nautilus DevOps Team needed to set up a private Virtual Network (VNet) with a subnet that isolates resources from external networks. A VM named `datacenter-priv-vm` was deployed within this VNet with an NSG that restricts SSH access to only traffic originating from within the VNet's own CIDR block — ensuring no public internet access to the VM.

---

## Requirements

| Parameter | Value |
|---|---|
| VNet Name | `datacenter-priv-vnet` |
| VNet CIDR | `10.0.0.0/16` |
| Subnet Name | `datacenter-priv-subnet` |
| Subnet CIDR | `10.0.1.0/24` |
| NSG Name | `datacenter-priv-nsg` |
| NSG Rule Source | `10.0.0.0/16` (VNet CIDR only) |
| NSG Rule Destination | `10.0.0.0/16` |
| NSG Rule Port | `22` (TCP) |
| NSG Rule Action | `Allow` |
| VM Name | `datacenter-priv-vm` |
| VM Size | `Standard_B1s` |
| OS Image | Ubuntu 22.04 LTS |
| Admin Username | `azureuser` |
| Public IP | None (private only) |
| Region | Central US |

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

### 2. Create the Private VNet and Subnet Together
```bash
az network vnet create \
    --resource-group kml_rg_main-af44574cd3db4043 \
    --name datacenter-priv-vnet \
    --location centralus \
    --address-prefix 10.0.0.0/16 \
    --subnet-name datacenter-priv-subnet \
    --subnet-prefix 10.0.1.0/24
```

> VNet and subnet can be created in a single command using `--subnet-name` and `--subnet-prefix` flags.

### 3. Create the NSG
```bash
az network nsg create \
    --resource-group kml_rg_main-af44574cd3db4043 \
    --name datacenter-priv-nsg \
    --location centralus
```

### 4. Add Inbound SSH Rule (VNet CIDR Only)
```bash
az network nsg rule create \
    --resource-group kml_rg_main-af44574cd3db4043 \
    --nsg-name datacenter-priv-nsg \
    --name allow-ssh-vnet \
    --protocol Tcp \
    --direction Inbound \
    --priority 100 \
    --source-address-prefix 10.0.0.0/16 \
    --source-port-range '*' \
    --destination-address-prefix 10.0.0.0/16 \
    --destination-port-range 22 \
    --access Allow
```

> Both source and destination are set to `10.0.0.0/16` — ensuring SSH is only allowed between resources within the VNet, completely blocking public internet access.

### 5. Associate NSG with Subnet
```bash
az network vnet subnet update \
    --resource-group kml_rg_main-af44574cd3db4043 \
    --vnet-name datacenter-priv-vnet \
    --name datacenter-priv-subnet \
    --network-security-group datacenter-priv-nsg
```

### 6. Generate SSH Key if Missing
```bash
[ -f ~/.ssh/id_rsa.pub ] || ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa -N ""
```

### 7. Create NIC Without Public IP
```bash
az network nic create \
    --resource-group kml_rg_main-af44574cd3db4043 \
    --name datacenter-priv-vm-nic \
    --location centralus \
    --vnet-name datacenter-priv-vnet \
    --subnet datacenter-priv-subnet \
    --network-security-group datacenter-priv-nsg
```

> **No `--public-ip-address` flag** — omitting it entirely ensures no public IP is assigned. Passing `--public-ip-address ""` causes an error.

### 8. Create the VM (No Public IP)
```bash
az vm create \
    --resource-group kml_rg_main-af44574cd3db4043 \
    --name datacenter-priv-vm \
    --location centralus \
    --image Ubuntu2204 \
    --size Standard_B1s \
    --admin-username azureuser \
    --ssh-key-values ~/.ssh/id_rsa.pub \
    --nics datacenter-priv-vm-nic \
    --os-disk-size-gb 30 \
    --storage-sku Standard_LRS
```

### 9. Verify All Resources
```bash
echo "=== VNet ==="
az network vnet show \
    --resource-group kml_rg_main-af44574cd3db4043 \
    --name datacenter-priv-vnet \
    --query "{Name:name, CIDR:addressSpace.addressPrefixes, Location:location}" \
    -o table

echo "=== Subnet ==="
az network vnet subnet show \
    --resource-group kml_rg_main-af44574cd3db4043 \
    --vnet-name datacenter-priv-vnet \
    --name datacenter-priv-subnet \
    --query "{Name:name, Prefix:addressPrefix, NSG:networkSecurityGroup.id}" \
    -o table

echo "=== NSG Rules ==="
az network nsg rule list \
    --resource-group kml_rg_main-af44574cd3db4043 \
    --nsg-name datacenter-priv-nsg \
    --query "[].{Name:name, Priority:priority, Source:sourceAddressPrefix, Dest:destinationAddressPrefix, Port:destinationPortRange, Access:access}" \
    -o table

echo "=== VM Private IP ==="
az vm list-ip-addresses \
    --resource-group kml_rg_main-af44574cd3db4043 \
    --name datacenter-priv-vm \
    -o table
```

---

## Issues Encountered & Fixes

| Issue | Cause | Fix |
|---|---|---|
| `InvalidResourceReference` on NIC create | Passed `--public-ip-address ""` (empty string) which Azure tried to resolve as a resource path | Omitted the `--public-ip-address` flag entirely |

---

## Verification Output

```
=== VNet ===
Name                   CIDR              Location
---------------------  ----------------  ----------
datacenter-priv-vnet   ['10.0.0.0/16']   centralus

=== Subnet ===
Name                     Prefix        NSG
-----------------------  ------------  --------------------------
datacenter-priv-subnet   10.0.1.0/24   .../datacenter-priv-nsg

=== NSG Rules ===
Name            Priority  Source        Dest          Port  Access
--------------  --------  ------------  ------------  ----  ------
allow-ssh-vnet  100       10.0.0.0/16   10.0.0.0/16   22    Allow
```

| Item | Value |
|---|---|
| VNet | `datacenter-priv-vnet` — `10.0.0.0/16` ✅ |
| Subnet | `datacenter-priv-subnet` — `10.0.1.0/24` ✅ |
| NSG | `datacenter-priv-nsg` — SSH from VNet CIDR only ✅ |
| Public IP | None — private VM only ✅ |
| VM | `datacenter-priv-vm` running ✅ |

---

## Key Concepts

**What Makes a VNet "Private"?**
A private VNet in Azure means resources inside it have no public IP addresses and no inbound access from the internet. All communication happens only within the VNet or through explicitly configured private peering/VPN connections. The NSG enforces this by only allowing traffic from the VNet CIDR block.

**Why Set Both Source AND Destination to `10.0.0.0/16`?**
The task explicitly required:
- Source: `10.0.0.0/16` — only VNet-internal traffic can initiate SSH
- Destination: `10.0.0.0/16` — traffic can only reach VNet-internal addresses

This double restriction ensures SSH communication is fully contained within the VNet — no external source can reach the VM and the VM can only receive SSH on its internal address range.

**Why Omit `--public-ip-address` Instead of Passing Empty String?**
Azure CLI interprets `--public-ip-address ""` as a reference to a public IP resource with an empty name, then tries to resolve it — causing `InvalidResourceReference`. Simply omitting the flag tells Azure not to assign any public IP at all, which is the correct approach for private VMs.

**NSG on Subnet vs NIC**
Attaching the NSG to the subnet (via `az network vnet subnet update`) applies the rules to all resources in that subnet, not just the VM's NIC. This is the recommended approach for consistent security across all resources in a private subnet.
