# Task 7 - Allocate a Public IP Address on Azure

## Overview

This task is part of the Nautilus DevOps team's incremental cloud migration strategy to Microsoft Azure. A Public IP address is a critical networking component that allows Azure resources such as Virtual Machines, Load Balancers, and Application Gateways to communicate with the internet. This task focuses on allocating a Public IP address that can later be associated with any Azure resource.

---

## Objectives

- Allocate a Public IP address named `devops-pip`

---

## Prerequisites

- Access to an active Azure subscription or lab credentials
- Azure CLI installed and authenticated
- An existing Resource Group

---

## Tools Used

| Tool | Purpose |
|------|---------|
| Azure CLI | Provisioning the Public IP via command line |
| Azure Portal | Alternative GUI-based provisioning |
| Microsoft Azure | Cloud platform |

---

## Steps

### Option A: Via Azure CLI

#### 1. Authenticate with Azure

```bash
az login
```

> For KodeKloud labs, credentials are pre-configured. Skip this step if already logged in.

#### 2. Identify the Resource Group

```bash
az group list --output table
```

Note the resource group name from the output.

#### 3. Create the Public IP Address

```bash
az network public-ip create \
  --name devops-pip \
  --resource-group <ResourceGroupName>
```

#### 4. Verify the Public IP was Created

```bash
az network public-ip show \
  --name devops-pip \
  --resource-group <ResourceGroupName> \
  --query "{Name:name, SKU:sku.name, AllocationMethod:publicIPAllocationMethod, IPAddress:ipAddress}" \
  -o table
```

---

### Option B: Via Azure Portal

1. Go to [portal.azure.com](https://portal.azure.com) and sign in
2. In the top search bar, type **"Public IP addresses"** and select it
3. Click **"+ Create"**
4. Fill in the details:
   - **Subscription:** leave as is
   - **Resource Group:** select the existing one
   - **Region:** leave as default
   - **Name:** `devops-pip`
   - **IP Version:** IPv4
   - **SKU:** Standard
   - **Tier:** Regional
   - **Assignment:** Static
5. Click **"Review + Create"** → then **"Create"**
6. Once deployed, confirm `devops-pip` appears under **Public IP addresses**

---

## Expected Output

```
Name        SKU       AllocationMethod    IPAddress
----------  --------  ------------------  ---------------
devops-pip  Standard  Static              <AssignedIP>
```

---

## Key Concepts

- **Public IP Address:** An IP address that is reachable from the internet. It allows inbound communication to Azure resources and outbound communication to the internet.
- **SKU (Standard vs Basic):** Standard SKU is zone-redundant, more secure by default, and supports availability zones. Basic SKU is older and being phased out.
- **Static vs Dynamic Allocation:**
  - **Static:** The IP address is assigned immediately and does not change even if the resource is stopped.
  - **Dynamic:** The IP address is assigned only when the resource starts and may change on restart.
- **pip:** A common Azure naming convention suffix standing for **P**ublic **I**P address.

---

## Notes

- A Public IP address alone does not expose any resource — it must be associated with a NIC, Load Balancer, or other resource.
- Standard SKU Public IPs are **Static** by default and are more secure as they are closed to inbound traffic unless explicitly allowed by an NSG.
- Public IPs incur a small cost in production environments when not associated with a running resource.
- Use meaningful naming conventions like `<project>-pip` for easier management at scale.

---

