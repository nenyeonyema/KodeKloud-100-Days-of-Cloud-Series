# Task 28 — Fix Broken Nginx VM (VNet, NSG, Routing and Nginx Setup)

## Overview

The Nautilus DevOps Team had a partially deployed VM named `datacenter-vm` in a public VNet named `datacenter-vnet` that was inaccessible from the internet. The task involved diagnosing and fixing multiple issues: attaching a pre-existing static public IP (`datacenter-pip`), configuring NSG rules to allow HTTP traffic on port 80, fixing a broken route table that was blocking all outbound internet traffic, and installing and starting Nginx.

---

## Requirements

| Parameter | Value |
|---|---|
| VM Name | `datacenter-vm` |
| VNet Name | `datacenter-vnet` |
| Public IP Name | `datacenter-pip` (pre-existing) |
| HTTP Port | 80 — open from internet |
| Web Server | Nginx — installed, enabled, running |
| Region | West US |

---

## Prerequisites

- Azure CLI installed and configured on the azure-client host
- `datacenter-vm` already exists but is misconfigured
- `datacenter-pip` already exists but is not attached to the VM

---

## Diagnosis Steps

### 1. Login to Azure
```bash
az login -u "<username>" -p "<password>"
az group list --output table
```

### 2. Inspect All Existing Resources
```bash
RG="kml_rg_main-e48d58d1dc864948"

# Check VM
az vm show --resource-group $RG --name datacenter-vm \
    --query "{Name:name, Size:hardwareProfile.vmSize, State:provisioningState}" \
    -o table

# Check public IP
az network public-ip show --resource-group $RG --name datacenter-pip \
    --query "{Name:name, IP:ipAddress, Allocation:publicIPAllocationMethod, AssociatedTo:ipConfiguration.id}" \
    -o table

# Check NIC
NIC_NAME=$(az vm show --resource-group $RG --name datacenter-vm \
    --query "networkProfile.networkInterfaces[0].id" -o tsv | xargs basename)

az network nic show --resource-group $RG --name $NIC_NAME \
    --query "{PublicIP:ipConfigurations[0].publicIPAddress.id, NSG:networkSecurityGroup.id}" \
    -o table

# Check NSG rules including defaults
NSG_NAME=$(az network nsg list --resource-group $RG --query "[0].name" -o tsv)
az network nsg rule list --resource-group $RG --nsg-name $NSG_NAME \
    --include-default \
    --query "[].{Name:name, Dir:direction, Dest:destinationAddressPrefix, Port:destinationPortRange, Access:access, Priority:priority}" \
    -o table

# Check route table
az network route-table list --resource-group $RG -o table

az network route-table route list \
    --resource-group $RG \
    --route-table-name datacenter-rtb \
    --query "[].{Name:name, Prefix:addressPrefix, NextHopType:nextHopType}" \
    -o table
```

### Findings

| Problem Found | Description |
|---|---|
| `datacenter-pip` not attached | Public IP existed but was not associated with the VM's NIC |
| No HTTP rule in NSG | Port 80 was not open — only SSH existed |
| NSG not attached to NIC | NSG existed but wasn't linked to the VM's NIC |
| `Block-Internet` route | Route table `datacenter-rtb` had a `0.0.0.0/0` route with next hop `None` blocking all outbound traffic |

---

## Fix Steps Performed

### 3. Attach datacenter-pip to the VM's NIC
```bash
RG="kml_rg_main-e48d58d1dc864948"
NIC_NAME=$(az vm show --resource-group $RG --name datacenter-vm \
    --query "networkProfile.networkInterfaces[0].id" -o tsv | xargs basename)

NIC_IP_CONFIG=$(az network nic show \
    --resource-group $RG --name $NIC_NAME \
    --query "ipConfigurations[0].name" -o tsv)

az network nic ip-config update \
    --resource-group $RG \
    --nic-name $NIC_NAME \
    --name $NIC_IP_CONFIG \
    --public-ip-address datacenter-pip

echo "Public IP attached!"
```

### 4. Add HTTP and SSH Rules to NSG
```bash
NSG_NAME=$(az network nsg list --resource-group $RG --query "[0].name" -o tsv)

az network nsg rule create \
    --resource-group $RG \
    --nsg-name $NSG_NAME \
    --name allow-http \
    --protocol Tcp \
    --direction Inbound \
    --priority 100 \
    --source-address-prefix Internet \
    --source-port-range '*' \
    --destination-address-prefix '*' \
    --destination-port-range 80 \
    --access Allow

az network nsg rule create \
    --resource-group $RG \
    --nsg-name $NSG_NAME \
    --name allow-ssh \
    --protocol Tcp \
    --direction Inbound \
    --priority 110 \
    --source-address-prefix Internet \
    --source-port-range '*' \
    --destination-address-prefix '*' \
    --destination-port-range 22 \
    --access Allow
```

### 5. Attach NSG to NIC
```bash
az network nic update \
    --resource-group $RG \
    --name $NIC_NAME \
    --network-security-group $NSG_NAME

echo "NSG attached to NIC!"
```

### 6. Also Attach NSG to Subnet
```bash
VNET_NAME=$(az network vnet list --resource-group $RG --query "[0].name" -o tsv)
SUBNET_NAME=$(az network vnet subnet list \
    --resource-group $RG --vnet-name $VNET_NAME \
    --query "[0].name" -o tsv)

az network vnet subnet update \
    --resource-group $RG \
    --vnet-name $VNET_NAME \
    --name $SUBNET_NAME \
    --network-security-group $NSG_NAME

echo "NSG attached to subnet!"
```

### 7. Fix the Blocking Route Table
```bash
# Delete the Block-Internet route that was blackholing all outbound traffic
az network route-table route delete \
    --resource-group $RG \
    --route-table-name datacenter-rtb \
    --name Block-Internet

# Add correct default internet route
az network route-table route create \
    --resource-group $RG \
    --route-table-name datacenter-rtb \
    --name internet-route \
    --address-prefix 0.0.0.0/0 \
    --next-hop-type Internet

echo "Internet route fixed!"
```

### 8. Start VM and Install Nginx via run-command
```bash
az vm start --resource-group $RG --name datacenter-vm
az vm wait --resource-group $RG --name datacenter-vm --updated

az vm run-command invoke \
    --resource-group $RG \
    --name datacenter-vm \
    --command-id RunShellScript \
    --scripts "
        sudo apt-get update -y && \
        sudo apt-get install -y nginx && \
        sudo systemctl enable nginx && \
        sudo systemctl start nginx && \
        sudo systemctl status nginx --no-pager | head -5 && \
        sudo ss -tlnp | grep :80
    "
```

> `az vm run-command invoke` was used instead of SSH because the SSH port was timing out while NSG issues were still being resolved. This command runs scripts directly on the VM via the Azure agent — no network access required.

### 9. Final HTTP Verification
```bash
PUBLIC_IP=$(az network public-ip show \
    --resource-group $RG \
    --name datacenter-pip \
    --query ipAddress -o tsv)

curl -s -o /dev/null -w "HTTP Status: %{http_code}\n" http://$PUBLIC_IP
```

---

## Issues Encountered & Fixes

| Issue | Cause | Fix |
|---|---|---|
| SSH connection timed out | NSG not attached to NIC | Attached NSG to both NIC and subnet |
| Nginx install failed (no internet) | `Block-Internet` route in `datacenter-rtb` blackholing all `0.0.0.0/0` outbound traffic | Deleted `Block-Internet` route, added `internet-route` with `NextHopType: Internet` |
| NAT Gateway creation blocked | Lab policy `Azure_KKE_free` disallows NAT Gateways | Solved by fixing the route table instead |
| Outbound NSG rule didn't help | NSG outbound `Allow Internet` rule was correct but route table overrides NSG for routing decisions | Route table fix was the actual solution |

---

## Verification Output

```
● nginx.service - A high performance web server
   Active: active (running)

LISTEN 0  511  0.0.0.0:80   0.0.0.0:*
LISTEN 0  511     [::]:80      [::]:*

HTTP Status: 200
```

| Item | Value |
|---|---|
| Public IP Attached | `datacenter-pip` → VM NIC ✅ |
| NSG Port 80 | Open from Internet ✅ |
| NSG Port 22 | Open from Internet ✅ |
| Nginx | Installed, enabled, running ✅ |
| HTTP Response | `200 OK` ✅ |

---

## Key Concepts

**Route Tables Override NSG for Routing Decisions**
A Network Security Group controls whether traffic is *allowed or denied*. A Route Table controls *where traffic is sent*. Even if the NSG has an outbound `Allow Internet` rule, a route table entry pointing `0.0.0.0/0` to `None` (blackhole) will silently drop all outbound packets before the NSG even evaluates them. Route table issues are harder to spot because the VM appears healthy from Azure's perspective.

**`Block-Internet` Route — A Classic Misconfiguration**
The route named `Block-Internet` had `address-prefix: 0.0.0.0/0` and `next-hop-type: None`. This is a deliberate blackhole route — any traffic destined for the internet had nowhere to go. Deleting it and replacing with `next-hop-type: Internet` restored outbound connectivity.

**Why `az vm run-command` Instead of SSH?**
`az vm run-command invoke` sends commands directly to the VM via the Azure VM Agent (a background process on every Azure VM) — completely bypassing the network stack. This is invaluable when SSH is unavailable due to NSG issues, and is the recommended way to troubleshoot or bootstrap VMs when network access is broken.

**NAT Gateway vs Route Table for Outbound Internet**
NAT Gateway is the preferred Azure solution for outbound internet from private subnets, but it was blocked by lab policy. Fixing the route table's default route (`0.0.0.0/0 → Internet`) achieves the same result for VMs that already have a public IP — the public IP provides the outbound SNAT automatically once the route is correct.
