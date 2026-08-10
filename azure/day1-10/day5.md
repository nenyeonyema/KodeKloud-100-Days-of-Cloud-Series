# Task 5 - Create a Virtual Network (VNet) on Azure

## Overview

This task involves creating a Virtual Network (VNet) on Microsoft Azure as part of an incremental cloud migration strategy for the Nautilus DevOps team. VNets serve as the foundational networking layer for provisioning various Azure services.

---

## Objectives

- Create a Virtual Network named `datacenter-vnet`
- Deploy it in the `centralus` region
- Configure the IPv4 CIDR block as `192.168.0.0/24`

---

## Prerequisites

- Access to an active Azure subscription or lab credentials
- Azure CLI installed and authenticated
- An existing Resource Group

---

## Tools Used

| Tool | Purpose |
|------|---------|
| Azure CLI | Provisioning the VNet via command line |
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

### 3. Create the Virtual Network

```bash
az network vnet create \
  --name datacenter-vnet \
  --location centralus \
  --address-prefix 192.168.0.0/24 \
  --resource-group <ResourceGroupName>
```

### 4. Verify the VNet was Created

```bash
az network vnet show \
  --name datacenter-vnet \
  --resource-group <ResourceGroupName> \
  --query "{Name:name, Location:location, AddressPrefix:addressSpace.addressPrefixes}" \
  -o table
```

---

## Expected Output

```
Name              Location    AddressPrefix
----------------  ----------  ----------------
datacenter-vnet   centralus   192.168.0.0/24
```

---

## Key Concepts

- **Virtual Network (VNet):** The fundamental building block for private networking in Azure. It enables Azure resources to securely communicate with each other, the internet, and on-premises networks.
- **CIDR Block:** Classless Inter-Domain Routing notation used to define the IP address range of the VNet. `192.168.0.0/24` provides 256 IP addresses.
- **Region:** The Azure data center location where the VNet is deployed. Resources within the same region communicate with lower latency.
- **Resource Group:** A logical container that holds related Azure resources for a solution.

---

## Notes

- A VNet is region-specific — resources in different regions cannot communicate directly through a VNet without peering.
- The `/24` CIDR provides 256 addresses (254 usable), suitable for small to medium workloads.
- Subnets can be added later to further segment the VNet address space.

---

## References

- [Azure Virtual Network Documentation](https://learn.microsoft.com/en-us/azure/virtual-network/)
- [az network vnet CLI Reference](https://learn.microsoft.com/en-us/cli/azure/network/vnet)
