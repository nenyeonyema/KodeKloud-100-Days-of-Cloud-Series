# Task 6 - Create a Virtual Network (VNet) with a Subnet on Azure

## Overview

This task is part of the Nautilus DevOps team's incremental cloud migration strategy to Microsoft Azure. Building on the foundational VNet creation, this task goes a step further by also provisioning a subnet within the VNet, enabling more granular network segmentation for services to be deployed later.

---

## Objectives

- Create a Virtual Network named `datacenter-vnet`
- Create a Subnet named `datacenter-subnet` within the VNet
- Deploy in the `westus` region
- Configure the IPv4 address range as `10.0.0.0/16`

---

## Prerequisites

- Access to an active Azure subscription or lab credentials
- Azure CLI installed and authenticated
- An existing Resource Group

---

## Tools Used

| Tool | Purpose |
|------|---------|
| Azure CLI | Provisioning the VNet and Subnet via command line |
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

### 3. Create the Virtual Network with Subnet

```bash
az network vnet create \
  --name datacenter-vnet \
  --location westus \
  --address-prefix 10.0.0.0/16 \
  --subnet-name datacenter-subnet \
  --subnet-prefix 10.0.0.0/24 \
  --resource-group <ResourceGroupName>
```

> The `--subnet-name` and `--subnet-prefix` flags allow you to create the VNet and its first subnet in a single command.

### 4. Verify the VNet was Created

```bash
az network vnet show \
  --name datacenter-vnet \
  --resource-group <ResourceGroupName> \
  --query "{Name:name, Location:location, AddressPrefix:addressSpace.addressPrefixes}" \
  -o table
```

### 5. Verify the Subnet was Created

```bash
az network vnet subnet list \
  --vnet-name datacenter-vnet \
  --resource-group <ResourceGroupName> \
  --output table
```

---

## Expected Output

**VNet:**
```
Name              Location    AddressPrefix
----------------  ----------  -------------
datacenter-vnet   westus      10.0.0.0/16
```

**Subnet:**
```
Name                 AddressPrefix    ProvisioningState
-------------------  ---------------  -------------------
datacenter-subnet    10.0.0.0/24      Succeeded
```

---

## Key Concepts

- **Virtual Network (VNet):** The fundamental building block for private networking in Azure. It enables Azure resources to securely communicate with each other, the internet, and on-premises networks.
- **Subnet:** A subdivision of a VNet's IP address range. Subnets allow you to segment the VNet into smaller networks, enabling better organization, security, and traffic control.
- **CIDR Block:** Classless Inter-Domain Routing notation used to define IP address ranges. `10.0.0.0/16` provides 65,536 IP addresses across the VNet, while `10.0.0.0/24` provides 256 addresses for the subnet.
- **Region:** The Azure data center location where resources are deployed. `westus` is a commonly used region for low-latency access in the western United States.
- **Resource Group:** A logical container that holds related Azure resources for a solution.

---

## Notes

- The VNet address space (`10.0.0.0/16`) must be large enough to contain all subnets within it.
- The subnet prefix (`10.0.0.0/24`) must fall within the VNet address space.
- Additional subnets can be added later using `az network vnet subnet create`.
- A `/16` CIDR is suitable for large workloads requiring many subnets and IP addresses.

---

