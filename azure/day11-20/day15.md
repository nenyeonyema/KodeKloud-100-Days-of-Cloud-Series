# Task 15 - Create a Network Security Group (NSG) with Inbound Rules on Azure

## Overview

This task is part of the Nautilus DevOps team's incremental cloud migration strategy to Microsoft Azure. Network Security Groups (NSGs) are a fundamental security layer in Azure that control inbound and outbound traffic to Azure resources. This task focuses on creating an NSG named `nautilus-nsg` and adding two inbound security rules to allow HTTP and SSH traffic.

---

## Objectives

- Create a Network Security Group named `nautilus-nsg`
- Add an inbound security rule named `Allow-HTTP` for HTTP traffic on port `80` with source CIDR `0.0.0.0/0`
- Add an inbound security rule named `Allow-SSH` for SSH traffic on port `22` with source CIDR `0.0.0.0/0`

---

## Prerequisites

- Access to an active Azure subscription or lab credentials
- Azure CLI installed and authenticated
- An existing Resource Group

---

## Tools Used

| Tool | Purpose |
|------|---------|
| Azure CLI | Creating the NSG and rules via command line |
| Microsoft Azure | Cloud platform |

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

### 3. Create the Network Security Group

```bash
az network nsg create \
  --name nautilus-nsg \
  --resource-group <ResourceGroupName>
```

### 4. Add the Allow-HTTP Inbound Rule

```bash
az network nsg rule create \
  --name Allow-HTTP \
  --nsg-name nautilus-nsg \
  --resource-group <ResourceGroupName> \
  --priority 100 \
  --protocol Tcp \
  --direction Inbound \
  --source-address-prefix 0.0.0.0/0 \
  --destination-port-range 80 \
  --access Allow
```

### 5. Add the Allow-SSH Inbound Rule

```bash
az network nsg rule create \
  --name Allow-SSH \
  --nsg-name nautilus-nsg \
  --resource-group <ResourceGroupName> \
  --priority 110 \
  --protocol Tcp \
  --direction Inbound \
  --source-address-prefix 0.0.0.0/0 \
  --destination-port-range 22 \
  --access Allow
```

### 6. Verify the NSG and Rules

```bash
az network nsg rule list \
  --nsg-name nautilus-nsg \
  --resource-group <ResourceGroupName> \
  --output table
```

---

## Expected Output

```
Name         Protocol    SourcePortRange    DestinationPortRange    SourceAddressPrefix    Access    Priority    Direction
-----------  ----------  -----------------  ----------------------  ---------------------  --------  ----------  -----------
Allow-HTTP   Tcp         *                  80                      0.0.0.0/0              Allow     100         Inbound
Allow-SSH    Tcp         *                  22                      0.0.0.0/0              Allow     110         Inbound
```

---

## NSG Rule Properties Explained

| Property | Description |
|----------|-------------|
| `--priority` | A number between 100 and 4096. Lower numbers are processed first. Each rule must have a unique priority within the NSG. |
| `--protocol` | The network protocol the rule applies to: `Tcp`, `Udp`, `Icmp`, or `*` (all). |
| `--direction` | Whether the rule applies to `Inbound` or `Outbound` traffic. |
| `--source-address-prefix` | The source IP range. `0.0.0.0/0` means all internet traffic. |
| `--destination-port-range` | The port or port range the rule applies to. |
| `--access` | Whether to `Allow` or `Deny` the matching traffic. |

---

## Key Concepts

- **Network Security Group (NSG):** A firewall-like resource in Azure that contains a list of security rules to allow or deny inbound and outbound network traffic to Azure resources.
- **Inbound Rules:** Control traffic coming into the resource (e.g. from the internet to a VM).
- **Outbound Rules:** Control traffic going out of the resource (e.g. from a VM to the internet).
- **Default Rules:** Every NSG comes with built-in default rules that cannot be deleted but can be overridden by custom rules with a lower priority number. Default rules include:
  - Allow all inbound traffic within the VNet
  - Allow inbound traffic from Azure Load Balancer
  - Deny all other inbound traffic
  - Allow all outbound traffic within the VNet and to the internet
  - Deny all other outbound traffic
- **Priority:** Rules are evaluated in priority order — lowest number first. Once a matching rule is found, processing stops. No two rules in the same NSG can have the same priority.
- **CIDR `0.0.0.0/0`:** Represents all IP addresses — essentially allowing traffic from anywhere on the internet.

---

## Notes

- An NSG must be **associated** with a subnet or NIC to take effect — creating it alone does not protect any resource.
- To associate the NSG with a subnet:
  ```bash
  az network vnet subnet update \
    --vnet-name <VNetName> \
    --name <SubnetName> \
    --resource-group <ResourceGroupName> \
    --network-security-group nautilus-nsg
  ```
- To associate the NSG with a NIC:
  ```bash
  az network nic update \
    --name <NICName> \
    --resource-group <ResourceGroupName> \
    --network-security-group nautilus-nsg
  ```
- In production environments, avoid using `0.0.0.0/0` as the source for SSH (port 22) — restrict it to specific trusted IP ranges to reduce attack surface.
- Priority numbers should be spaced apart (e.g. 100, 110, 120) to allow room for inserting new rules between existing ones later.

---

