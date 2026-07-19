# Azure Task 31 — Python Web App Deployment with Azure App Service

## Overview

This task demonstrates deploying a Python-based web application on Azure using Azure App Service. A new App Service Plan is created with a Linux runtime environment, and a web app is deployed with the Python runtime stack. The web app is tagged for identification and verified to be in a Running state after creation.

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│              Azure App Service                          │
│              West US                                    │
│                                                         │
│   ┌─────────────────────────────────────────────────┐   │
│   │         nautilus-learn-python                   │   │
│   │         App Service Plan                        │   │
│   │         SKU: Basic B1 | OS: Linux               │   │
│   │                                                 │   │
│   │   ┌─────────────────────────────────────────┐   │   │
│   │   │         nautilus-webapp                 │   │   │
│   │   │         Runtime: Python 3.11            │   │   │
│   │   │         Publish: Code                   │   │   │
│   │   │         State: Running                  │   │   │
│   │   │                                         │   │   │
│   │   │  Tags:                                  │   │   │
│   │   │  Name: WebAppLearning                   │   │   │
│   │   │  Environment: Dev                       │   │   │
│   │   └─────────────────────────────────────────┘   │   │
│   └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

---

## Prerequisites

- Azure CLI installed and authenticated
- Resource group: `kml_rg_main-4938febae8fb4b31`
- Region: West US (East US had insufficient quota — see notes)

---

## Step 1 — Login

```bash
az login -u kk_lab_user_main-4938febae8fb4b31@azurefreekmlprod.onmicrosoft.com -p "s^sssr86"
```

---

## Step 2 — Get Resource Group

```bash
az group list --query "[0].name" -o tsv
```

Output: `kml_rg_main-4938febae8fb4b31`

---

## Step 3 — Create the App Service Plan

```bash
az appservice plan create \
  --name nautilus-learn-python \
  --resource-group kml_rg_main-4938febae8fb4b31 \
  --location westus \
  --sku B1 \
  --is-linux
```

**Flags explained:**
- `--sku B1` — Basic B1 tier, suitable for dev/test workloads
- `--is-linux` — Sets the OS to Linux, required for Python runtime
- `--location westus` — East US was attempted first but hit VM quota limits (0 Total VMs allowed); West US had available quota

---

## Step 4 — Create the Web App

```bash
az webapp create \
  --name nautilus-webapp \
  --resource-group kml_rg_main-4938febae8fb4b31 \
  --plan nautilus-learn-python \
  --runtime "PYTHON:3.11" \
  --tags Name=WebAppLearning Environment=Dev
```

**Flags explained:**
- `--runtime "PYTHON:3.11"` — Sets Python 3.11 as the runtime stack
- `--tags` — Applies Name and Environment tags inline at creation time
- Publish mode defaults to **Code** (as required) when no container/Docker flags are passed

---

## Step 5 — Verify Running State

```bash
az webapp show \
  --name nautilus-webapp \
  --resource-group kml_rg_main-4938febae8fb4b31 \
  --query "{name:name, state:state, location:location}" \
  -o table
```

Output:
```
Name             State    Location
---------------  -------  ----------
nautilus-webapp  Running  West US
```

✅ Web app is Running.

---

## Key Concepts

**App Service Plan vs Web App**
These are two separate resources in Azure App Service:

| Resource | Role |
|---|---|
| App Service Plan | Defines the underlying compute (region, OS, SKU, scaling) |
| Web App | The actual application running on top of the plan |

Multiple web apps can share one App Service Plan, splitting the compute cost.

**Why `--is-linux` on the plan?**
Python runtime on Azure App Service requires a Linux-based App Service Plan. Windows plans do not support Python. The OS is set at the **plan level**, not the web app level — so it must be specified when creating the plan.

**Why West US instead of East US?**
The lab subscription had a quota limit of 0 Total VMs in East US, blocking the B1 plan creation. West US had available quota. The resource group was in East US but Azure allows App Service Plans to be created in a different region from the resource group.

**Tags at creation time**
Tags can be applied inline during `az webapp create` using `--tags Key=Value`. This is more efficient than a separate `az tag` command and ensures resources are tagged from the moment they exist.

---

## Troubleshooting

**East US quota error:**
```
Operation cannot be completed without additional quota.
Current Limit (Total VMs): 0
[O```
→ Switch to a region with available quota e.g. `--location westus`

**Check available runtimes:**
```bash
az webapp list-runtimes --os-type linux
```

---

## Outcome

| Task | Result |
|---|---|
| App Service Plan nautilus-learn-python created (Linux, Basic B1) | ✅ |
| Web App nautilus-webapp created (Python 3.11, Code publish) | ✅ |
| Tags applied (Name=WebAppLearning, Environment=Dev) | ✅ |
| Web App state → Running | ✅ |
