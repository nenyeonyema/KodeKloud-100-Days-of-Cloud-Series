# Azure Task 50 — Application Gateway Load Balancing Across Two VMs

## Overview

This task demonstrates setting up an Azure Application Gateway (AGW) as a Layer 7 load balancer distributing HTTP traffic across two Nginx VMs serving different content. The task required precise resource naming to pass validation — a key lesson learned after an initial failure due to a frontend IP configuration name mismatch. The AGW must be created using the Azure REST API directly since the lab subscription policy only allows the Basic SKU, which the Azure CLI does not support as a valid `--sku` value.

---

## Architecture

```
                         Internet
                             |
                             | HTTP :80
                             v
                  +---------------------+
                  |   datacenter-apgw   |
                  |   Basic SKU AGW     |
                  |   East US           |
                  |                     |
                  |  Frontend IP:       |
                  |  datacenter-apgw-ip |
                  |  57.162.163.109     |
                  +----------+----------+
                             |
                   +---------v------------------+
                   |   datacenter-backendpool   |
                   +---------+------------------+
                             |
              +--------------+--------------+
              |                             |
    +---------v---------+       +-----------v-------+
    |   datacenter-vm1  |       |   datacenter-vm2  |
    |   10.0.1.4        |       |   10.0.1.5        |
    |   Nginx :80       |       |   Nginx :80       |
    |   Version 1       |       |   Version 2       |
    +-------------------+       +-------------------+

  Listener:      datacenter-listener (HTTP :80)
  HTTP Settings: datacenter-http-settings (port 80)
  Routing Rule:  datacenter-routing-rule (Basic)
  Frontend IP:   datacenter-apgw-ip (EXACT NAME - critical)

  NSG Rules (datacenter-nsg):
  +-- Allow-SSH        -> port 22          (priority 110)
  +-- Allow-HTTP       -> port 80          (priority 100)
  +-- Allow-AGW-Probe  -> 65200-65535      (priority 120)

  VNet: datacenter-vnet (10.0.0.0/16)
  +-- datacenter-subnet      -> 10.0.1.0/24  (VMs)
  +-- datacenter-apgw-subnet -> 10.0.2.0/24  (AGW dedicated)
```

---

## Prerequisites

- Azure CLI installed and authenticated
- Resource group: kml_rg_main-68db37650c404061
- SSH key pair generated at ~/.ssh/id_rsa
- Region: East US

---

## Step 1 — Login

```bash
az login -u kk_lab_user_main-68db37650c404061@azurefreekmlprod.onmicrosoft.com -p "3gwrbp$-"
az group list --query "[0].name" -o tsv
```

---

## Step 2 — Create VNet and Both Subnets

```bash
az network vnet create \
  --resource-group kml_rg_main-68db37650c404061 \
  --name datacenter-vnet \
  --location eastus \
  --address-prefix 10.0.0.0/16 \
  --subnet-name datacenter-subnet \
  --subnet-prefix 10.0.1.0/24

az network vnet subnet create \
  --resource-group kml_rg_main-68db37650c404061 \
  --vnet-name datacenter-vnet \
  --name datacenter-apgw-subnet \
  --address-prefix 10.0.2.0/24
```

Note: AGW requires a dedicated subnet — no other resources can share it.

---

## Step 3 — Create NSG with All Required Rules Upfront

```bash
az network nsg create \
  --resource-group kml_rg_main-68db37650c404061 \
  --name datacenter-nsg \
  --location eastus

az network nsg rule create \
  --resource-group kml_rg_main-68db37650c404061 \
  --nsg-name datacenter-nsg \
  --name Allow-HTTP \
  --priority 100 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --destination-port-ranges 80

az network nsg rule create \
  --resource-group kml_rg_main-68db37650c404061 \
  --nsg-name datacenter-nsg \
  --name Allow-SSH \
  --priority 110 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --destination-port-ranges 22

az network nsg rule create \
  --resource-group kml_rg_main-68db37650c404061 \
  --nsg-name datacenter-nsg \
  --name Allow-AGW-Probe \
  --priority 120 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --source-address-prefixes GatewayManager \
  --destination-port-ranges 65200-65535
```

---

## Step 4 — Generate SSH Key

```bash
ls ~/.ssh/id_rsa.pub 2>/dev/null && echo "exists" || ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa -N ""
```

---

## Step 5 — Create Both VMs

```bash
az vm create \
  --resource-group kml_rg_main-68db37650c404061 \
  --name datacenter-vm1 \
  --image Ubuntu2204 \
  --size Standard_B1s \
  --admin-username azureuser \
  --ssh-key-values ~/.ssh/id_rsa.pub \
  --vnet-name datacenter-vnet \
  --subnet datacenter-subnet \
  --nsg datacenter-nsg \
  --public-ip-sku Standard \
  --location eastus \
  --storage-sku Standard_LRS \
  --os-disk-size-gb 30

az vm create \
  --resource-group kml_rg_main-68db37650c404061 \
  --name datacenter-vm2 \
  --image Ubuntu2204 \
  --size Standard_B1s \
  --admin-username azureuser \
  --ssh-key-values ~/.ssh/id_rsa.pub \
  --vnet-name datacenter-vnet \
  --subnet datacenter-subnet \
  --nsg datacenter-nsg \
  --public-ip-sku Standard \
  --location eastus \
  --storage-sku Standard_LRS \
  --os-disk-size-gb 30
```

VM outputs:
- datacenter-vm1: private IP 10.0.1.4
- datacenter-vm2: private IP 10.0.1.5

---

## Step 6 — Create AGW Public IP with Exact Name

```bash
az network public-ip create \
  --resource-group kml_rg_main-68db37650c404061 \
  --name datacenter-apgw-ip \
  --sku Standard \
  --allocation-method Static \
  --location eastus
```

Assigned IP: 57.162.163.109

---

## Step 7 — Install Nginx on Both VMs

```bash
# VM1 - Version 1
ssh azureuser@<VM1_PUBLIC_IP> "sudo apt-get update -y && sudo apt-get install -y nginx && echo 'Welcome to KKE Labs:Version 1' | sudo tee /var/www/html/index.html && sudo systemctl start nginx"

# VM2 - Version 2
ssh azureuser@<VM2_PUBLIC_IP> "sudo apt-get update -y && sudo apt-get install -y nginx && echo 'Welcome to KKE Labs:Version 2' | sudo tee /var/www/html/index.html && sudo systemctl start nginx"
```

Verify:
```bash
curl http://<VM1_PUBLIC_IP>   # Welcome to KKE Labs:Version 1
curl http://<VM2_PUBLIC_IP>   # Welcome to KKE Labs:Version 2
```

---

## Step 8 — Create Application Gateway via az rest (Basic SKU)

The lab policy only allows Basic SKU but the CLI does not accept Basic as a valid --sku value. Use az rest to call the ARM API directly:

```bash
az rest --method PUT \
  --url "https://management.azure.com/subscriptions/f0c3bcdd-5ce2-4fa0-8cf3-41559747512b/resourceGroups/kml_rg_main-68db37650c404061/providers/Microsoft.Network/applicationGateways/datacenter-apgw?api-version=2023-04-01" \
  --body '{
    "location": "eastus",
    "properties": {
      "sku": { "name": "Basic", "tier": "Basic", "capacity": 1 },
      "gatewayIPConfigurations": [{
        "name": "appGatewayIpConfig",
        "properties": {
          "subnet": {
            "id": "/subscriptions/f0c3bcdd-5ce2-4fa0-8cf3-41559747512b/resourceGroups/kml_rg_main-68db37650c404061/providers/Microsoft.Network/virtualNetworks/datacenter-vnet/subnets/datacenter-apgw-subnet"
          }
        }
      }],
      "frontendIPConfigurations": [{
        "name": "datacenter-apgw-ip",
        "properties": {
          "publicIPAddress": {
            "id": "/subscriptions/f0c3bcdd-5ce2-4fa0-8cf3-41559747512b/resourceGroups/kml_rg_main-68db37650c404061/providers/Microsoft.Network/publicIPAddresses/datacenter-apgw-ip"
          }
        }
      }],
      "frontendPorts": [{ "name": "port_80", "properties": { "port": 80 } }],
      "backendAddressPools": [{
        "name": "datacenter-backendpool",
        "properties": {
          "backendAddresses": [
            { "ipAddress": "10.0.1.4" },
            { "ipAddress": "10.0.1.5" }
          ]
        }
      }],
      "backendHttpSettingsCollection": [{
        "name": "datacenter-http-settings",
        "properties": {
          "port": 80,
          "protocol": "Http",
          "cookieBasedAffinity": "Disabled",
          "requestTimeout": 20
        }
      }],
      "httpListeners": [{
        "name": "datacenter-listener",
        "properties": {
          "frontendIPConfiguration": {
            "id": "/subscriptions/f0c3bcdd-5ce2-4fa0-8cf3-41559747512b/resourceGroups/kml_rg_main-68db37650c404061/providers/Microsoft.Network/applicationGateways/datacenter-apgw/frontendIPConfigurations/datacenter-apgw-ip"
          },
          "frontendPort": {
            "id": "/subscriptions/f0c3bcdd-5ce2-4fa0-8cf3-41559747512b/resourceGroups/kml_rg_main-68db37650c404061/providers/Microsoft.Network/applicationGateways/datacenter-apgw/frontendPorts/port_80"
          },
          "protocol": "Http"
        }
      }],
      "requestRoutingRules": [{
        "name": "datacenter-routing-rule",
        "properties": {
          "ruleType": "Basic",
          "priority": 100,
          "httpListener": {
            "id": "/subscriptions/f0c3bcdd-5ce2-4fa0-8cf3-41559747512b/resourceGroups/kml_rg_main-68db37650c404061/providers/Microsoft.Network/applicationGateways/datacenter-apgw/httpListeners/datacenter-listener"
          },
          "backendAddressPool": {
            "id": "/subscriptions/f0c3bcdd-5ce2-4fa0-8cf3-41559747512b/resourceGroups/kml_rg_main-68db37650c404061/providers/Microsoft.Network/applicationGateways/datacenter-apgw/backendAddressPools/datacenter-backendpool"
          },
          "backendHttpSettings": {
            "id": "/subscriptions/f0c3bcdd-5ce2-4fa0-8cf3-41559747512b/resourceGroups/kml_rg_main-68db37650c404061/providers/Microsoft.Network/applicationGateways/datacenter-apgw/backendHttpSettingsCollection/datacenter-http-settings"
          }
        }
      }]
    }
  }'
```

---

## Step 9 — Wait for Succeeded State

```bash
watch -n 15 "az network application-gateway show \
  --resource-group kml_rg_main-68db37650c404061 \
  --name datacenter-apgw \
  --query '{name:name, state:provisioningState, operational:operationalState}' \
  -o table"
```

Wait until: `State: Succeeded` and `Operational: Running`

---

## Step 10 — Verify Load Balancing

```bash
curl http://57.162.163.109
curl http://57.162.163.109
curl http://57.162.163.109
curl http://57.162.163.109
```

Output:
```
Welcome to KKE Labs:Version 1
Welcome to KKE Labs:Version 2
Welcome to KKE Labs:Version 2
Welcome to KKE Labs:Version 1
```

---

## Key Concepts

**Why the first attempt failed**
The validation script checked for a frontend IP configuration named exactly `nautilus-apgw-ip`. When creating the AGW via the portal, the public IP was accidentally named `nautilus-agw-ip` (missing the `p`) and the frontend IP config name was not set correctly. The fix — always create the public IP via CLI first with the exact name, then reference it during AGW creation.

**Why ports 65200-65535 must be open**
Azure uses these ports for AGW infrastructure health probes from the `GatewayManager` service tag. Without this NSG rule the AGW cannot verify backend health — backends show as Unhealthy and the AGW returns 502. This rule must be added BEFORE the AGW is created to avoid 502 errors on first test.

**Why az rest instead of az network application-gateway create**
The lab policy enforces Basic SKU only. The CLI --sku flag does not accept Basic as a valid value. az rest calls the ARM API directly, bypassing CLI validation while satisfying the policy.

**AGW provisioning time**
Basic SKU AGW via az rest takes 10-15 minutes to fully provision. The az rest command returns immediately with provisioningState: Updating — do NOT test with curl until the state shows Succeeded and operationalState shows Running. Polling with watch is the right approach.

**Resource naming is critical for lab validation**
KodeKloud validation scripts check exact resource names. Every component must match:

| Resource | Required Name |
|---|---|
| Public IP | datacenter-apgw-ip |
| Frontend IP config | datacenter-apgw-ip |
| Backend pool | datacenter-backendpool |
| HTTP settings | datacenter-http-settings |
| Listener | datacenter-listener |
| Routing rule | datacenter-routing-rule |

A single character difference (nautilus-agw-ip vs nautilus-apgw-ip) causes validation failure even when the AGW is functionally working.

---

## Lessons Learned

1. Always create the public IP via CLI with the exact name before creating the AGW — never let the portal auto-generate names.
2. Add the AGW probe NSG rule (65200-65535 from GatewayManager) before creating the AGW to avoid 502 errors.
3. Use az rest for Basic SKU AGW — the CLI --sku flag does not accept Basic.
4. Wait for provisioningState: Succeeded before testing — az rest returns immediately while Azure continues provisioning in the background.
5. Exact resource names matter — lab validation scripts check every component name precisely.
6. A 502 from the AGW means backends are unhealthy — check NSG probe ports first before investigating further.

---

## Outcome

| Task | Result |
|---|---|
| datacenter-vnet created (10.0.0.0/16) | Done |
| datacenter-subnet (10.0.1.0/24) created | Done |
| datacenter-apgw-subnet (10.0.2.0/24) created | Done |
| datacenter-nsg with HTTP, SSH, AGW probe rules | Done |
| datacenter-vm1 created with Nginx (Version 1) | Done |
| datacenter-vm2 created with Nginx (Version 2) | Done |
| datacenter-apgw-ip public IP created (57.162.163.109) | Done |
| datacenter-apgw created (Basic SKU via az rest) | Done |
| Frontend IP config named exactly datacenter-apgw-ip | Done |
| Both VMs in datacenter-backendpool | Done |
| Load balancing verified (Version 1 and Version 2) | Done |
| Lab validation passed | Done |
