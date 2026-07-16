# Azure VNet Peering — Task 35

## Overview

The Nautilus DevOps team has been tasked with demonstrating the use of VNet Peering to enable communication between two VNets. One VNet will be a private VNet that contains a private Azure VM, while the other will be a public VNet containing a publicly accessible Azure VM.
> 1) Existing Azure Resources:
Public VM: datacenter-pub-vm is already in the public VNet.
Private VNet and VM: datacenter-priv-vnet and datacenter-priv-vm exist in the private VNet with its subnet: datacenter-priv-subnet.
> 2) Create VNet Peering:
Create a VNet Peering between the Public VNet and Private VNet.
 VNet Peering Name: datacenter-pub-to-priv-peering.
> 3) Test the Connection:
SSH into the public VM and verify that you can ping the private VM.
>
---

## Architecture

```
┌─────────────────────────────┐        VNet Peering        ┌─────────────────────────────┐
│      datacenter-pub-vnet    │◄──────────────────────────►│     datacenter-priv-vnet    │
│                             │                            │                             │
│  ┌───────────────────────┐  │                            │  ┌───────────────────────┐  │
│  │   datacenter-pub-vm   │  │                            │  │   datacenter-priv-vm  │  │
│  │   Public IP:          │  │                            │  │   Private IP:         │  │
│  │   20.106.253.80       │  │                            │  │   10.1.1.4            │  │
│  └───────────────────────┘  │                            │  └───────────────────────┘  │
└─────────────────────────────┘                            └─────────────────────────────┘
```

---

## Prerequisites

- Azure CLI installed and authenticated
- Two existing VNets: `datacenter-pub-vnet` and `datacenter-priv-vnet`
- Two existing VMs: `datacenter-pub-vm` (public) and `datacenter-priv-vm` (private)
- Resource group: `kml_rg_main-809edcb6c2434839`
- Region: East US

---

## Steps Performed

### 1. Discover Existing Resources

List all VNets in the resource group:

```bash
az network vnet list \
  --resource-group kml_rg_main-809edcb6c2434839 \
  --query "[].{name:name, id:id, addressSpace:addressSpace.addressPrefixes}" \
  -o table
```

Retrieve VNet IDs needed for peering:

```bash
az network vnet show \
  --resource-group kml_rg_main-809edcb6c2434839 \
  --name datacenter-pub-vnet \
  --query "id" -o tsv

az network vnet show \
  --resource-group kml_rg_main-809edcb6c2434839 \
  --name datacenter-priv-vnet \
  --query "id" -o tsv
```

---

### 2. Create VNet Peering (Public → Private)

VNet Peering must be created in **both directions** for bidirectional traffic flow.

**Forward peering (public → private):**

```bash
az network vnet peering create \
  --resource-group kml_rg_main-809edcb6c2434839 \
  --name datacenter-pub-to-priv-peering \
  --vnet-name datacenter-pub-vnet \
  --remote-vnet /subscriptions/f0c3bcdd-5ce2-4fa0-8cf3-41559747512b/resourceGroups/kml_rg_main-809edcb6c2434839/providers/Microsoft.Network/virtualNetworks/datacenter-priv-vnet \
  --allow-vnet-access
```

**Reverse peering (private → public):**

```bash
az network vnet peering create \
  --resource-group kml_rg_main-809edcb6c2434839 \
  --name datacenter-priv-to-pub-peering \
  --vnet-name datacenter-priv-vnet \
  --remote-vnet /subscriptions/f0c3bcdd-5ce2-4fa0-8cf3-41559747512b/resourceGroups/kml_rg_main-809edcb6c2434839/providers/Microsoft.Network/virtualNetworks/datacenter-pub-vnet \
  --allow-vnet-access
```

---

### 3. Verify Peering Status

```bash
az network vnet peering list \
  --resource-group kml_rg_main-809edcb6c2434839 \
  --vnet-name datacenter-pub-vnet \
  --query "[].{name:name, state:peeringState}" -o table
```

Expected output:

```
Name                            State
------------------------------  ---------
datacenter-pub-to-priv-peering  Connected
```

---

### 4. Retrieve VM IP Addresses

```bash
# Private VM's private IP
az vm list-ip-addresses \
  --resource-group kml_rg_main-809edcb6c2434839 \
  --name datacenter-priv-vm \
  --query "[].virtualMachine.network.privateIpAddresses" -o tsv

# Public VM's public IP
az vm list-ip-addresses \
  --resource-group kml_rg_main-809edcb6c2434839 \
  --name datacenter-pub-vm \
  --query "[].virtualMachine.network.publicIpAddresses[0].ipAddress" -o tsv
```

| VM | IP Type | Address |
|---|---|---|
| datacenter-pub-vm | Public IP | 20.106.253.80 |
| datacenter-priv-vm | Private IP | 10.1.1.4 |

---

### 5. Test Connectivity

SSH into the public VM:

```bash
ssh azureuser@20.106.253.80
```

Ping the private VM from within the public VM:

```bash
ping -c 4 10.1.1.4
```

Expected output:

```
PING 10.1.1.4 (10.1.1.4) 56(84) bytes of data.
64 bytes from 10.1.1.4: icmp_seq=1 ttl=64 time=1.57 ms
64 bytes from 10.1.1.4: icmp_seq=2 ttl=64 time=0.789 ms
64 bytes from 10.1.1.4: icmp_seq=3 ttl=64 time=0.881 ms
64 bytes from 10.1.1.4: icmp_seq=4 ttl=64 time=0.665 ms

--- 10.1.1.4 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3086ms
```

✅ 0% packet loss confirms successful VNet peering and private connectivity.

---

## Key Concepts

**Why two peering directions?**
Azure VNet Peering is non-transitive and directional. Creating a peering on only one side results in one-way traffic at best, or no traffic at all. Both `pub → priv` and `priv → pub` peerings must exist and show `Connected` for full bidirectional communication.

**Why use VNet Peering instead of a VPN?**
VNet Peering uses the Azure backbone network — it is lower latency, higher bandwidth, and simpler to configure than a VPN Gateway for VNets within the same region or subscription.

**`--allow-vnet-access` flag**
This flag enables traffic between the peered VNets. Without it, the peering link exists but traffic is blocked.

---

## Outcome

| Check | Result |
|---|---|
| `datacenter-pub-to-priv-peering` created | ✅ |
| `datacenter-priv-to-pub-peering` created | ✅ |
| Both peerings in Connected state | ✅ |
| Ping from public VM to private VM (10.1.1.4) | ✅ 0% packet loss |
