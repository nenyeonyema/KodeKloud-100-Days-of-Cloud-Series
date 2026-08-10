# Task 20 - Modify and Deploy an ARM Template for a Virtual Network on Azure

## Overview

This task is part of the Nautilus DevOps team's incremental cloud migration strategy to Microsoft Azure. Azure Resource Manager (ARM) templates are JSON files that define the infrastructure and configuration for Azure resources. This task focuses on modifying an existing ARM template to update a virtual network's name, address prefix, and tags, then deploying it using the Azure CLI.

---

## Objectives

- Change the name and `displayName` tag of the virtual network to `arm-vnet-xfusion`
- Update the `addressPrefixes` to `192.168.0.0/16`
- Add a new tag named `Environment` with value `KKE-xfusion`
- Deploy the modified ARM template using the Azure CLI

---

## Prerequisites

- Access to an active Azure subscription or lab credentials
- Azure CLI installed and authenticated
- An existing Resource Group
- An existing ARM template located at `/root/arm-templates/vnet-deployment-template.json`

---

## Tools Used

| Tool | Purpose |
|------|---------|
| Azure CLI | Deploying the ARM template |
| sed | Modifying the ARM template from the command line |
| Microsoft Azure | Cloud platform |

---

## Original Template

```json
{
    "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {},
    "functions": [],
    "variables": {},
    "resources": [
        {
            "name": "virtualNetwork1",
            "type": "Microsoft.Network/virtualNetworks",
            "apiVersion": "2023-11-01",
            "location": "[resourceGroup().location]",
            "tags": {
                "displayName": "virtualNetwork1"
            },
            "properties": {
                "addressSpace": {
                    "addressPrefixes": [
                        "10.10.10.0/24"
                    ]
                }
            }
        }
    ],
    "outputs": {
    }
}
```

---

## Modified Template

```json
{
    "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {},
    "functions": [],
    "variables": {},
    "resources": [
        {
            "name": "arm-vnet-xfusion",
            "type": "Microsoft.Network/virtualNetworks",
            "apiVersion": "2023-11-01",
            "location": "[resourceGroup().location]",
            "tags": {
                "displayName": "arm-vnet-xfusion",
                "Environment": "KKE-xfusion"
            },
            "properties": {
                "addressSpace": {
                    "addressPrefixes": [
                        "192.168.0.0/16"
                    ]
                }
            }
        }
    ],
    "outputs": {
    }
}
```

---

## Steps

### 1. Authenticate with Azure

```bash
az login
```

> For KodeKloud labs, credentials are pre-configured. Skip this step if already logged in.

### 2. Identify the Resource Group

```bash
az group list --query '[].name' --output table | grep 'kml'
```

### 3. View the Current Template

```bash
cat /root/arm-templates/vnet-deployment-template.json
```

### 4. Make All Required Changes Using sed

```bash
# Change the VNet name
sed -i 's/"name": "virtualNetwork1"/"name": "arm-vnet-xfusion"/' \
  /root/arm-templates/vnet-deployment-template.json

# Update the displayName tag
sed -i 's/"displayName": "virtualNetwork1"/"displayName": "arm-vnet-xfusion"/' \
  /root/arm-templates/vnet-deployment-template.json

# Update the address prefix
sed -i 's/"10.10.10.0\/24"/"192.168.0.0\/16"/' \
  /root/arm-templates/vnet-deployment-template.json

# Add the Environment tag
sed -i 's/"displayName": "arm-vnet-xfusion"/"displayName": "arm-vnet-xfusion",\n                "Environment": "KKE-xfusion"/' \
  /root/arm-templates/vnet-deployment-template.json
```

### 5. Verify the Changes

```bash
cat /root/arm-templates/vnet-deployment-template.json
```

Confirm all four changes are reflected correctly before deploying.

### 6. Deploy the ARM Template

```bash
az deployment group create \
  --resource-group <ResourceGroupName> \
  --template-file /root/arm-templates/vnet-deployment-template.json
```

### 7. Verify the Deployment

```bash
az network vnet list \
  --resource-group <ResourceGroupName> \
  --query "[].{Name:name, AddressPrefix:addressSpace.addressPrefixes, Tags:tags}" \
  -o table
```

---

## Expected Output

**Deployment:**
```json
{
  "properties": {
    "provisioningState": "Succeeded",
    "outputResources": [
      {
        "id": "/subscriptions/.../resourceGroups/.../providers/Microsoft.Network/virtualNetworks/arm-vnet-xfusion"
      }
    ]
  }
}
```

**VNet verification:**
```
Name              AddressPrefix       Tags
----------------  ------------------  -----------------------------------------
arm-vnet-xfusion  ['192.168.0.0/16']  {'displayName': 'arm-vnet-xfusion', 'Environment': 'KKE-xfusion'}
```

---

## ARM Template Structure Explained

| Section | Purpose |
|---------|---------|
| `$schema` | Defines the ARM template schema version |
| `contentVersion` | Version of the template for tracking changes |
| `parameters` | Input values that can be passed at deployment time |
| `functions` | Custom functions used within the template |
| `variables` | Reusable values within the template |
| `resources` | The Azure resources to be deployed |
| `outputs` | Values returned after deployment completes |

---

## Key Concepts

- **ARM Template:** A JSON file that declaratively defines Azure infrastructure. It describes the desired end state and Azure handles the deployment order and dependencies automatically.
- **Idempotency:** ARM templates are idempotent — deploying the same template multiple times results in the same outcome without duplicating resources.
- **`az deployment group create`:** The Azure CLI command used to deploy an ARM template to a specific resource group.
- **`sed` (Stream Editor):** A Unix command-line tool for parsing and transforming text. Used here to make targeted string replacements in the JSON template file.
- **`[resourceGroup().location]`:** An ARM template function that dynamically resolves to the location of the resource group being deployed to, avoiding hardcoding a region.
- **Tags in ARM Templates:** Tags are defined as key-value pairs within the `tags` section of a resource definition and are deployed alongside the resource.

---

## Notes

- Always verify the template with `cat` after making `sed` changes and before deploying to catch any formatting errors.
- ARM templates can be validated without deploying using:
  ```bash
  az deployment group validate \
    --resource-group <ResourceGroupName> \
    --template-file /root/arm-templates/vnet-deployment-template.json
  ```
- For more complex template edits, use a text editor like `nano` or `vim` instead of `sed`:
  ```bash
  nano /root/arm-templates/vnet-deployment-template.json
  ```
- ARM templates are being gradually supplemented by **Bicep**, a domain-specific language that compiles to ARM JSON and is easier to read and write.
- Deployment history can be viewed with:
  ```bash
  az deployment group list \
    --resource-group <ResourceGroupName> \
    --output table
  ```

---

