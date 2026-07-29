# Azure Task 43 — Application Gateway with Nginx Backend VM

## Overview

This task demonstrates setting up an Azure Application Gateway (AGW) in front of a Virtual Machine running Nginx. The AGW acts as a layer 7 load balancer and traffic manager, routing HTTP requests from a public IP to the backend VM. The task covers NSG configuration, VNet/subnet design, VM provisioning with user data, and AGW deployment using the Azure REST API to work around a lab policy constraint.

---

## Architecture

```
                         Internet
                             |
                             | HTTP :80
                             v
                  +---------------------+
                  |    datacenter-agw   |
                  |   (Basic SKU AGW)   |
                  |                     |
                  |  Frontend IP:       |
                  |  datacenter-agw-ip  |
                  |  40.65.56.216       |
                  +----------+----------+
                             |
                   +---------v------------------+
                   | datacenter-backendpool     |
                   +---------+------------------+
                             | HTTP :80
                    +--------v--------+
                    |  datacenter-vm  |
                    |  West US        |
                    |  10.0.1.4       |
                    |  Nginx :80      |
                    +-----------------+

  Listener:      datacenter-listener (HTTP :80)
  HTTP Settings: datacenter-http-settings (port 80)
  Routing Rule:  datacenter-routing-rule (Basic)

  NSG Rules (datacenter-nsg):
  +-- Allow-SSH        -> port 22          (priority 90)
  +-- Allow-HTTP       -> port 80          (priority 100)
  +-- Allow-AGW-Probe  -> 65200-65535      (priority 110)

  VNet: datacenter-vnet (10.0.0.0/16)
  +-- datacenter-vm-subnet  -> 10.0.1.0/24  (VM)
  +-- datacenter-agw-subnet -> 10.0.2.0/24  (AGW dedicated)
```

---

## Prerequisites

- Azure CLI installed and authenticated
- Resource group: kml_rg_main-9417a10ed8ba4d3e
- SSH key pair generated at ~/.ssh/id_rsa
- Region: West US

---

## Step 1 — Login

```bash
az login -u kk_lab_user_main-9417a10ed8ba4d3e@azurefreekmlprod.onmicrosoft.com -p "PMYjy3vN"
```

---

## Step 2 — Generate SSH Key Pair

```bash
ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa -N ""
cat ~/.ssh/id_rsa.pub
```

---

## Step 3 — Create NSG and Allow-HTTP Rule

```bash
az network nsg create \
  --resource-group kml_rg_main-9417a10ed8ba4d3e \
  --name datacenter-nsg \
  --location westus

az network nsg rule create \
  --resource-group kml_rg_main-9417a10ed8ba4d3e \
  --nsg-name datacenter-nsg \
  --name Allow-HTTP \
  --priority 100 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --destination-port-ranges 80
```

---

## Step 4 — Create VNet and Subnets

```bash
az network vnet create \
  --resource-group kml_rg_main-9417a10ed8ba4d3e \
  --name datacenter-vnet \
  --location westus \
  --address-prefix 10.0.0.0/16 \
  --subnet-name datacenter-vm-subnet \
  --subnet-prefix 10.0.1.0/24

az network vnet subnet create \
  --resource-group kml_rg_main-9417a10ed8ba4d3e \
  --vnet-name datacenter-vnet \
  --name datacenter-agw-subnet \
  --address-prefix 10.0.2.0/24
```

Note: AGW requires its own dedicated subnet — no other resources can share it.

---

## Step 5 — Create Public IP for AGW

```bash
az network public-ip create \
  --resource-group kml_rg_main-9417a10ed8ba4d3e \
  --name datacenter-agw-ip \
  --sku Standard \
  --allocation-method Static \
  --location westus
```

Assigned IP: 40.65.56.216

---

## Step 6 — Create VM with Nginx User Data

```bash
cat > /tmp/userdata.sh << 'EOF'
#!/bin/bash
apt-get update -y
apt-get install -y nginx
systemctl start nginx
systemctl enable nginx
EOF

az vm create \
  --resource-group kml_rg_main-9417a10ed8ba4d3e \
  --name datacenter-vm \
  --image Ubuntu2204 \
  --size Standard_B1s \
  --admin-username azureuser \
  --ssh-key-values ~/.ssh/id_rsa.pub \
  --storage-sku Standard_LRS \
  --vnet-name datacenter-vnet \
  --subnet datacenter-vm-subnet \
  --nsg datacenter-nsg \
  --public-ip-sku Standard \
  --custom-data /tmp/userdata.sh \
  --location westus
```

VM result:
- Private IP: 10.0.1.4
- Public IP:  172.185.29.245
- State:      VM running

---

## Step 7 — Create Application Gateway via REST API

The lab subscription policy only allows Basic SKU for Application Gateway, but the Azure CLI does not list Basic as a valid --sku value. The workaround is to call the Azure Resource Manager REST API directly using az rest, which bypasses the CLI SKU validation while satisfying the lab policy.

```bash
az rest --method PUT \
  --url "https://management.azure.com/subscriptions/f0c3bcdd-5ce2-4fa0-8cf3-41559747512b/resourceGroups/kml_rg_main-9417a10ed8ba4d3e/providers/Microsoft.Network/applicationGateways/datacenter-agw?api-version=2023-04-01" \
  --body '{
    "location": "westus",
    "properties": {
      "sku": { "name": "Basic", "tier": "Basic", "capacity": 1 },
      "gatewayIPConfigurations": [{
        "name": "appGatewayIpConfig",
        "properties": {
          "subnet": {
            "id": "/subscriptions/f0c3bcdd-5ce2-4fa0-8cf3-41559747512b/resourceGroups/kml_rg_main-9417a10ed8ba4d3e/providers/Microsoft.Network/virtualNetworks/datacenter-vnet/subnets/datacenter-agw-subnet"
          }
        }
      }],
      "frontendIPConfigurations": [{
        "name": "datacenter-agw-ip",
        "properties": {
          "publicIPAddress": {
            "id": "/subscriptions/f0c3bcdd-5ce2-4fa0-8cf3-41559747512b/resourceGroups/kml_rg_main-9417a10ed8ba4d3e/providers/Microsoft.Network/publicIPAddresses/datacenter-agw-ip"
          }
        }
      }],
      "frontendPorts": [{ "name": "port_80", "properties": { "port": 80 } }],
      "backendAddressPools": [{
        "name": "datacenter-backendpool",
        "properties": { "backendAddresses": [{ "ipAddress": "10.0.1.4" }] }
      }],
      "backendHttpSettingsCollection": [{
        "name": "datacenter-http-settings",
        "properties": { "port": 80, "protocol": "Http", "cookieBasedAffinity": "Disabled", "requestTimeout": 20 }
      }],
      "httpListeners": [{
        "name": "datacenter-listener",
        "properties": {
          "frontendIPConfiguration": { "id": ".../frontendIPConfigurations/datacenter-agw-ip" },
          "frontendPort": { "id": ".../frontendPorts/port_80" },
          "protocol": "Http"
        }
      }],
      "requestRoutingRules": [{
        "name": "datacenter-routing-rule",
        "properties": {
          "ruleType": "Basic",
          "priority": 100,
          "httpListener": { "id": ".../httpListeners/datacenter-listener" },
          "backendAddressPool": { "id": ".../backendAddressPools/datacenter-backendpool" },
          "backendHttpSettings": { "id": ".../backendHttpSettingsCollection/datacenter-http-settings" }
        }
      }]
    }
  }'
```

AGW deployed with sku.name: Basic and sku.tier: Basic

---

## Step 8 — Add Missing NSG Rules

After deployment, SSH and AGW health probe traffic were blocked by the NSG. Two additional rules were required:

```bash
az network nsg rule create \
  --resource-group kml_rg_main-9417a10ed8ba4d3e \
  --nsg-name datacenter-nsg \
  --name Allow-SSH \
  --priority 90 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --destination-port-ranges 22

az network nsg rule create \
  --resource-group kml_rg_main-9417a10ed8ba4d3e \
  --nsg-name datacenter-nsg \
  --name Allow-AGW-Probe \
  --priority 110 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --source-address-prefixes GatewayManager \
  --destination-port-ranges 65200-65535
```

---

## Step 9 — Verify Nginx via AGW Public IP

```bash
curl http://40.65.56.216
```

Output:
```
<!DOCTYPE html>
<html>
<head><title>Welcome to nginx!</title></head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and working.</p>
</body>
</html>
```

Nginx is accessible via the Application Gateway public IP.

---

## Key Concepts

**Why does AGW need a dedicated subnet?**
Azure Application Gateway requires an exclusive subnet — no other resources (VMs, NICs, etc.) can share it. This is a hard Azure requirement; deploying AGW into a shared subnet will fail.

**Why ports 65200-65535 must be open on the NSG**
Azure uses ports 65200-65535 for AGW infrastructure communication and health probes between the AGW control plane (GatewayManager service tag) and the gateway instances. Without this rule, the AGW cannot verify backend health and no traffic will flow through — even if Nginx is running perfectly on the VM.

**Why use az rest instead of az network application-gateway create**
The lab subscription policy enforced Basic SKU only, but the Azure CLI --sku flag does not accept Basic as a valid value (only Standard_Small, Standard_v2, etc.). Using az rest to call the ARM API directly allows passing Basic as the SKU value, satisfying both the policy and the deployment requirement. This is a useful pattern whenever the CLI lags behind API capabilities or conflicts with policy constraints.

**Application Gateway vs Load Balancer**

| Feature            | Load Balancer     | Application Gateway  |
|--------------------|-------------------|----------------------|
| OSI Layer          | Layer 4 (TCP/UDP) | Layer 7 (HTTP/HTTPS) |
| Routing basis      | IP and port       | URL path, host header|
| SSL termination    | No                | Yes                  |
| WAF support        | No                | Yes (WAF SKU)        |
| Best for           | Any TCP traffic   | Web applications     |

**NSG rules required for this setup**

| Rule           | Priority | Port        | Source         | Purpose                    |
|----------------|----------|-------------|----------------|----------------------------|
| Allow-SSH      | 90       | 22          | Any            | Admin access to VM         |
| Allow-HTTP     | 100      | 80          | Any            | Web traffic to VM          |
| Allow-AGW-Probe| 110      | 65200-65535 | GatewayManager | AGW health probe (mandatory)|

---

## Lessons Learned

1. Always add the AGW probe ports (65200-65535 from GatewayManager) to the VM's NSG — without this the AGW backend health check fails and no traffic flows.
2. The --os-disk-sku flag is not recognized in this Azure CLI version; use --storage-sku Standard_LRS instead.
3. When the CLI's --sku options conflict with subscription policy, use az rest to call the ARM API directly.
4. SSH access (port 22) must be explicitly added to the NSG since only port 80 was configured at creation time.

---

## Outcome

| Task                                                  | Result |
|-------------------------------------------------------|--------|
| datacenter-nsg created with Allow-HTTP rule           | Done   |
| datacenter-vnet created (10.0.0.0/16)                 | Done   |
| datacenter-vm-subnet (10.0.1.0/24) created            | Done   |
| datacenter-agw-subnet (10.0.2.0/24) created           | Done   |
| datacenter-agw-ip public IP created (40.65.56.216)    | Done   |
| datacenter-vm created with Nginx user data            | Done   |
| datacenter-agw deployed (Basic SKU via az rest)       | Done   |
| datacenter-backendpool pointing to 10.0.1.4           | Done   |
| datacenter-http-settings configured (port 80)         | Done   |
| datacenter-listener configured (HTTP :80)             | Done   |
| datacenter-routing-rule configured                    | Done   |
| AGW probe NSG rule added (65200-65535)                | Done   |
| curl http://40.65.56.216 returns Nginx welcome page   | Done   |
