# Azure Task 4: Create a Virtual Network

## Overview

In this task, I created an Azure Virtual Network (VNet) for the Nautilus DevOps environment.

The VNet was created in the `southcentralus` region with an IPv4 address range. Since the task allowed any valid IPv4 CIDR block, the VNet was configured with `10.0.0.0/16`.

| Configuration | Value |
|---|---|
| VNet Name | `nautilus-vnet` |
| Region | `southcentralus` |
| Address Space | `10.0.0.0/16` |
| Address Type | IPv4 |

---

## Prerequisites

- Azure subscription
- Azure CLI installed
- Access to the Azure client/landing host
- Valid Azure lab credentials
- Existing Azure resource group

---

* 1 Retrieve Azure Credentials

The temporary lab credentials can be retrieved from the Azure client host using:

showcreds

* 2 Log in to Azure
Authenticate using Azure CLI:

```
az login
```
Verify the active Azure account and subscription:

```
az account show
```
* 3 Find the Existing Resource Group
The task requires using the existing resource group.

List the available resource groups:

```
az group list \
  --query "[].name" \
  --output table
```
To identify the lab resource group:

```
az group list \
  --query "[].name" \
  --output table | grep "kml"
```
Set the resource group as a shell variable:

RG="<resource-group>"
For example:
```
RG="kml_rg_main-xxxxxxxx"
```
* 4 Create the Virtual Network
Create the VNet using Azure CLI:
```
az network vnet create \
  --resource-group "$RG" \
  --name nautilus-vnet \
  --location southcentralus \
  --address-prefixes 10.0.0.0/16
```
The command creates:

VNet: nautilus-vnet

Region: southcentralus

IPv4 address space: 10.0.0.0/16

* 5 Verify the VNet

Check that the VNet was created successfully:

```
az network vnet show \
  --resource-group "$RG" \
  --name nautilus-vnet \
  --output table
```

Verify the name, location, and address space directly:

```
az network vnet show \
  --resource-group "$RG" \
  --name nautilus-vnet \
  --query "{Name:name,Location:location,AddressSpace:addressSpace.addressPrefixes}"
```

Expected result:
```
{
  "Name": "nautilus-vnet",
  "Location": "southcentralus",
  "AddressSpace": [
    "10.0.0.0/16"
  ]
}
```

* 6 List the VNet
The VNet can also be confirmed by listing all VNets in the resource group:

```
az network vnet list \
  --resource-group "$RG" \
  --output table
```

Expected output should contain:
```
Name             Location
---------------  -------------
nautilus-vnet    southcentralus
Final Verification
Requirement	Expected Value	Status
VNet Name	nautilus-vnet	✅
Region	southcentralus	✅
IPv4 Address Space	10.0.0.0/16	✅
VNet Created	Yes	✅
```

**Result
The nautilus-vnet Virtual Network was successfully created in the southcentralus region with the IPv4 address space:
```
10.0.0.0/16
```
The VNet is now available for deploying and networking Azure resources within the Nautilus environment.

Key Takeaways
This task demonstrated how to create and verify an Azure Virtual Network using the Azure CLI.

The main command used was:

```
az network vnet create \
  --resource-group "$RG" \
  --name nautilus-vnet \
  --location southcentralus \
  --address-prefixes 10.0.0.0/16
```
A VNet provides the foundational network boundary for Azure resources such as virtual machines, subnets, network interfaces, and other services.
