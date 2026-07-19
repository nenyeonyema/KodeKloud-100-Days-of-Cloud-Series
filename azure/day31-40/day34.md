# Azure Task 34 — Fix VM Connectivity Issue

## 📋 Task Overview

The Nautilus DevOps team encountered an issue with an Azure VM named `devops-vm` where package installation was failing due to a connectivity issue. The task required identifying the root cause and implementing a fix to restore normal operations.

---

## 🔍 Problem Statement

- **VM Name:** `devops-vm`
- **Issue:** Unable to install packages via `apt-get`
- **Error:** Connection timeout when reaching `azure.archive.ubuntu.com`
- **Region:** East US

---

## 🛠️ Investigation Steps

### Step 1 — Get VM Details

```bash
az group list --query "[].name" --output table
```

```bash
az vm show \
  --resource-group <resource-group> \
  --name devops-vm \
  --show-details \
  --query "{PublicIP:publicIps,PrivateIP:privateIps,Status:powerState}" \
  --output table
```

**Finding:** VM was running with public IP `172.190.28.166`.

---

### Step 2 — Reproduce the Issue

SSH into the VM:

```bash
ssh -i /root/.ssh/id_rsa azureuser@172.190.28.166
```

Run apt update:

```bash
sudo apt-get update
```

**Output:**
```
Err:1 http://azure.archive.ubuntu.com/ubuntu jammy InRelease
  Could not connect to azure.archive.ubuntu.com:80 (52.147.219.192), connection timed out
```

**Finding:** All outbound HTTP connections were timing out, confirming an outbound connectivity block.

---

### Step 3 — Identify the Root Cause

List NSGs in the resource group:

```bash
az network nsg list \
  --resource-group <resource-group> \
  --query "[].{Name:name}" \
  --output table
```

Inspect NSG rules:

```bash
az network nsg show \
  --resource-group <resource-group> \
  --name devops-nsg \
  --query "securityRules[].{Name:name,Direction:direction,Access:access,Priority:priority,Port:destinationPortRange}" \
  --output table
```

**Finding:** The NSG `devops-nsg` had the following rules:

| Name | Direction | Access | Priority | Port |
|---|---|---|---|---|
| Allow-SSH | Inbound | Allow | 100 | 22 |
| Allow-HTTP | Inbound | Allow | 110 | 80 |
| **Block-All-Outbound** | **Outbound** | **Deny** | **200** | **\*** |

The `Block-All-Outbound` rule at priority 200 was denying **all outbound traffic**, preventing the VM from reaching the Ubuntu package repositories on port 80.

---

## ✅ Solution

Add a new outbound **Allow** rule with a **lower priority number** (higher precedence) than 200, specifically permitting HTTP (port 80) and HTTPS (port 443) outbound traffic:

```bash
az network nsg rule create \
  --resource-group <resource-group> \
  --nsg-name devops-nsg \
  --name Allow-HTTP-HTTPS-Outbound \
  --direction Outbound \
  --access Allow \
  --priority 100 \
  --protocol Tcp \
  --destination-port-ranges 80 443 \
  --destination-address-prefixes Internet
```

---

## ✔️ Verification

After applying the fix, re-run `apt-get update` on the VM:

```bash
sudo apt-get update
```

**Result:**
```
Fetched 46.2 MB in 10s (4689 kB/s)
Reading package lists... Done
```

Package installation restored successfully.

---

## 📖 Key Concepts

### Azure NSG Priority Rules
In Azure Network Security Groups, rules are evaluated in **priority order** — lower number = higher priority. When two rules conflict:

```
Priority 100 (Allow outbound 80/443)  ← evaluated first → ALLOW
Priority 200 (Deny all outbound)      ← never reached for ports 80/443
```

By creating an Allow rule at priority 100, we ensured it is evaluated **before** the Deny-All rule at priority 200, selectively permitting only the traffic needed for package management while keeping the rest of the deny policy in place.

### Why Not Delete the Block Rule?
Deleting `Block-All-Outbound` entirely would open all outbound traffic, which may violate the team's security policy. Adding a specific allow rule at a higher priority is the correct approach — it maintains the principle of least privilege by only opening the ports required for package installation.

---

## 📝 NSG Rules After Fix

| Name | Direction | Access | Priority | Port |
|---|---|---|---|---|
| Allow-SSH | Inbound | Allow | 100 | 22 |
| Allow-HTTP | Inbound | Allow | 110 | 80 |
| **Allow-HTTP-HTTPS-Outbound** | **Outbound** | **Allow** | **100** | **80, 443** |
| Block-All-Outbound | Outbound | Deny | 200 | * |

---

## 🔧 Tools Used

- Azure CLI (`az`)
- Azure Network Security Groups (NSG)
- SSH
- Ubuntu `apt-get`
