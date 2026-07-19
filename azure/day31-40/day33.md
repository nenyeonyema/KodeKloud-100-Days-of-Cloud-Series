# Azure Task 33 — Azure Load Balancer Setup with Nginx Backend VM

## Overview

This task demonstrates setting up an Azure Load Balancer in front of a Virtual Machine running an Nginx web server. The Load Balancer distributes incoming HTTP traffic on port 80 to the backend VM, with a health probe monitoring VM availability. An NSG inbound rule is added to allow HTTP traffic to reach the VM.

---

## Architecture

```
                         Internet
                             │
                             │ HTTP :80
                             ▼
                  ┌─────────────────────┐
                  │    nautilus-lb       │
                  │   (Standard LB)     │
                  │                     │
                  │  Frontend IP:       │
                  │  nautilus-lb-ip     │
                  │  (Public Static IP) │
                  └──────────┬──────────┘
                             │
                   ┌─────────▼──────────┐
                   │ nautilus-backend-  │
                   │      pool          │
                   └─────────┬──────────┘
                             │
                    ┌────────▼────────┐
                    │  nautilus-vm    │
                    │  East US        │
                    │  Nginx :80      │
                    │                 │
                    │  NSG:           │
                    │  Allow HTTP 80  │
                    └─────────────────┘

  Health Probe: nautilus-health-probe → TCP :80
  LB Rule: nautilus-lb-rule → :80 → :80
```

---

## Prerequisites

- Azure CLI installed and authenticated
- Resource group: `kml_rg_main-485785a0940a4a04`
- `nautilus-vm` already exists in East US running Nginx
- VM NIC: `nautilus-vmVMNic`
- NSG: `nautilus-vmNSG`
- IP Config name: `ipconfignautilus-vm`

---

## Step 1 — Login

```bash
az login -u kk_lab_user_main-485785a0940a4a04@azurefreekmlprod.onmicrosoft.com -p "Q%HHHT9@"
```

---

## Step 2 — Discover Existing VM Resources

```bash
# Get resource group
az group list --query "[0].name" -o tsv

# Get VM NIC ID
az vm show \
  --resource-group kml_rg_main-485785a0940a4a04 \
  --name nautilus-vm \
  --query "networkProfile.networkInterfaces[0].id" -o tsv

# Get NIC details
az network nic show \
  --ids <NIC_ID> \
  --query "{nicName:name, subnetId:ipConfigurations[0].subnet.id, ipConfigName:ipConfigurations[0].name, nsg:networkSecurityGroup.id}" \
  -o json
```

Output:
```json
{
  "ipConfigName": "ipconfignautilus-vm",
  "nicName": "nautilus-vmVMNic",
  "nsg": ".../networkSecurityGroups/nautilus-vmNSG",
  "subnetId": ".../virtualNetworks/nautilus-vmVNET/subnets/nautilus-vmSubnet"
}
```

---

## Step 3 — Create the Public IP

```bash
az network public-ip create \
  --resource-group kml_rg_main-485785a0940a4a04 \
  --name nautilus-lb-ip \
  --sku Standard \
  --allocation-method Static \
  --location eastus
```

**Why Standard SKU?**
Standard Load Balancer requires a Standard SKU public IP. Basic SKU public IPs are incompatible with Standard Load Balancers.

---

## Step 4 — Create the Load Balancer with Frontend IP and Backend Pool

```bash
az network lb create \
  --resource-group kml_rg_main-485785a0940a4a04 \
  --name nautilus-lb \
  --location eastus \
  --sku Standard \
  --frontend-ip-name nautilus-lb-ip \
  --public-ip-address nautilus-lb-ip \
  --backend-pool-name nautilus-backend-pool
```

This single command creates three resources at once:
- The Load Balancer (`nautilus-lb`)
- The frontend IP configuration (`nautilus-lb-ip`)
- The backend pool (`nautilus-backend-pool`)

---

## Step 5 — Add the VM NIC to the Backend Pool

```bash
az network nic ip-config address-pool add \
  --resource-group kml_rg_main-485785a0940a4a04 \
  --nic-name nautilus-vmVMNic \
  --ip-config-name ipconfignautilus-vm \
  --lb-name nautilus-lb \
  --address-pool nautilus-backend-pool
```

This registers the VM's NIC IP configuration with the backend pool, making the VM a target for load-balanced traffic.

---

## Step 6 — Create the Health Probe

```bash
az network lb probe create \
  --resource-group kml_rg_main-485785a0940a4a04 \
  --lb-name nautilus-lb \
  --name nautilus-health-probe \
  --protocol Tcp \
  --port 80
```

The health probe checks TCP connectivity on port 80 every 15 seconds (default). If a VM fails the probe, it is removed from the backend pool until it recovers.

---

## Step 7 — Create the Load Balancer Rule

```bash
az network lb rule create \
  --resource-group kml_rg_main-485785a0940a4a04 \
  --lb-name nautilus-lb \
  --name nautilus-lb-rule \
  --protocol Tcp \
  --frontend-port 80 \
  --backend-port 80 \
  --frontend-ip-name nautilus-lb-ip \
  --backend-pool-name nautilus-backend-pool \
  --probe-name nautilus-health-probe
```

This rule routes all TCP traffic arriving on frontend port 80 to backend pool members on port 80, using the health probe to determine VM availability.

---

## Step 8 — Add NSG Inbound Rule for HTTP

```bash
az network nsg rule create \
  --resource-group kml_rg_main-485785a0940a4a04 \
  --nsg-name nautilus-vmNSG \
  --name Allow-HTTP \
  --priority 100 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --destination-port-ranges 80 \
  --source-address-prefixes '*' \
  --destination-address-prefixes '*'
```

Without this rule, the NSG blocks HTTP traffic even though the Load Balancer is correctly configured.

---

## Step 9 — Verify

```bash
az network lb show \
  --resource-group kml_rg_main-485785a0940a4a04 \
  --name nautilus-lb -o table

az network public-ip show \
  --resource-group kml_rg_main-485785a0940a4a04 \
  --name nautilus-lb-ip \
  --query "ipAddress" -o tsv
```

Then test the Nginx page via the LB public IP:
```bash
curl http://<LB_PUBLIC_IP>
```

---

## Key Concepts

**Why discover the VM's NIC and subnet before creating the LB?**
The backend pool association requires the exact NIC name and IP configuration name. Guessing these values causes the command to fail — always discover first.

**Load Balancer components and their relationship**

```
Load Balancer
├── Frontend IP Config  → entry point (public IP)
├── Backend Pool        → target VMs (via NIC association)
├── Health Probe        → monitors VM health on a port
└── LB Rule             → maps frontend port → backend port using probe
```

All four must exist and be linked for traffic to flow end-to-end.

**NSG and Load Balancer are independent**
The Load Balancer operates at the network layer and does not bypass NSG rules. Even with a perfectly configured LB, if the VM's NSG blocks port 80, traffic will not reach Nginx. Both must allow the traffic.

**Standard vs Basic Load Balancer**

| Feature | Basic | Standard |
|---|---|---|
| SKU requirement | Basic public IP | Standard public IP |
| Availability Zones | No | Yes |
| SLA | None | 99.99% |
| Recommended for | Dev/test | Production |

---

## Outcome

| Task | Result |
|---|---|
| Public IP nautilus-lb-ip created (Standard, Static) | ✅ |
| Load Balancer nautilus-lb created | ✅ |
| Frontend IP config nautilus-lb-ip configured | ✅ |
| Backend pool nautilus-backend-pool created | ✅ |
| nautilus-vm NIC added to backend pool | ✅ |
| Health probe nautilus-health-probe created (TCP :80) | ✅ |
| LB rule nautilus-lb-rule created (:80 → :80) | ✅ |
| NSG inbound rule Allow-HTTP added (port 80) | ✅ |
