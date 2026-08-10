# Task 12 - Add a Tag to a Virtual Machine on Azure

## Overview

This task is part of the Nautilus DevOps team's incremental cloud migration strategy to Microsoft Azure. During the migration, the team identified a virtual machine that was not tagged properly. Tags are key-value pairs that help organize, track, and manage Azure resources. This task focuses on adding the tag `Environment=dev` to the virtual machine named `xfusion-vm`.

---

## Objectives

- Add the tag `Environment=dev` to the virtual machine named `xfusion-vm`

---

## Prerequisites

- Access to an active Azure subscription or lab credentials
- Azure CLI installed and authenticated
- An existing Resource Group
- An existing VM named `xfusion-vm`

---

## Tools Used

| Tool | Purpose |
|------|---------|
| Azure CLI | Adding tags to the VM via command line |
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

### 3. Verify the Current Tags on the VM

```bash
az vm show \
  --name xfusion-vm \
  --resource-group <ResourceGroupName> \
  --query "tags" \
  -o table
```

### 4. Add the Tag to the VM

```bash
az vm update \
  --name xfusion-vm \
  --resource-group <ResourceGroupName> \
  --set tags.Environment=dev
```

### 5. Verify the Tag was Added

```bash
az vm show \
  --name xfusion-vm \
  --resource-group <ResourceGroupName> \
  --query "tags" \
  -o table
```

---

## Expected Output

**After tagging:**
```
Environment
-----------
dev
```

---

## Key Concepts

- **Tags:** Key-value pairs that can be applied to Azure resources for organizing, filtering, and tracking purposes. Tags are metadata and do not affect resource behavior.
- **Tag Use Cases:**
  - **Cost Management:** Group resources by project or department to track spending
  - **Environment Identification:** Label resources as `dev`, `staging`, or `prod`
  - **Automation:** Use tags to trigger automated scripts or policies
  - **Governance:** Enforce tagging policies using Azure Policy
- **Tag Limits:** Each Azure resource can have up to 50 tags. Tag names are case-insensitive but tag values are case-sensitive.
- **`--set` flag:** Used with `az vm update` to set or update a specific property on a resource without replacing the entire resource configuration.

---

## Notes

- Adding a tag with `--set tags.<key>=<value>` will **add** the tag without removing existing tags.
- To replace **all** tags on a resource use `az resource tag` with `--tags` instead.
- Tags can be applied to most Azure resources including VMs, VNets, storage accounts, and resource groups.
- Tagging at the resource group level does **not** automatically apply tags to resources within it — each resource must be tagged individually unless using Azure Policy.
- To add multiple tags at once:
  ```bash
  az vm update \
    --name xfusion-vm \
    --resource-group <ResourceGroupName> \
    --set tags.Environment=dev tags.Project=nautilus tags.Owner=devops
  ```

---

