# Task 2 - Create an Azure Virtual Machine (VM) via the Azure Portal

## Overview

This task is part of the Nautilus DevOps team's incremental cloud migration strategy to Microsoft Azure. Virtual Machines are the core compute resource in Azure, allowing teams to run workloads in the cloud just as they would on physical servers. This task focuses on creating a Linux VM using the Azure Portal with specific configurations including image, size, storage, and network security settings.

---

## Objectives

- Create a Virtual Machine named `devops-vm` in the `eastus` region
- Use the existing Resource Group
- Use the **Ubuntu 24.04 LTS** image
- Set the VM size to `Standard_B1s`
- Attach a default Network Security Group (NSG) that allows inbound SSH (port 22)
- Attach a **30 GB Standard HDD** storage disk
- Verify SSH access to the VM after creation

---

## Prerequisites

- Access to an active Azure subscription or lab credentials
- A web browser to access the Azure Portal
- An existing Resource Group

---

## Tools Used

| Tool | Purpose |
|------|---------|
| Azure Portal | Creating the VM via the GUI |
| SSH | Verifying connectivity to the VM |

---

## Steps

### 1. Sign in to the Azure Portal

Go to [portal.azure.com](https://portal.azure.com) and sign in with your credentials.

### 2. Navigate to Virtual Machines

In the top search bar, type **"Virtual Machines"** and select it from the results.

### 3. Create a New VM

Click **"+ Create"** → Select **"Azure Virtual Machine"**.

### 4. Configure the Basics Tab

Fill in the following fields:

| Field | Value |
|-------|-------|
| Subscription | Leave as default |
| Resource Group | Select the existing resource group |
| Virtual Machine Name | `devops-vm` |
| Region | `(US) East US` |
| Availability Options | No infrastructure redundancy required |
| Image | `Ubuntu Server 24.04 LTS` |
| VM Architecture | x64 |
| Size | `Standard_B1s` (click "See all sizes" if not visible) |
| Authentication Type | SSH public key or Password |
| Username | `azureuser` (default) |
| SSH Public Key / Password | Generate new key pair or set a password |

Click **"Next: Disks"**.

### 5. Configure the Disks Tab

| Field | Value |
|-------|-------|
| OS Disk Type | **Standard HDD** |
| OS Disk Size | **30 GB** (select "Change size" and set to 30 GiB) |

Leave all other disk settings as default.

Click **"Next: Networking"**.

### 6. Configure the Networking Tab

| Field | Value |
|-------|-------|
| Virtual Network | Leave as default (auto-created) |
| Subnet | Leave as default |
| Public IP | Leave as default (auto-created) |
| NIC Network Security Group | **Basic** |
| Public Inbound Ports | **Allow selected ports** |
| Select Inbound Ports | **SSH (22)** |

Click **"Next: Management"** → leave all as default.

Click **"Next: Monitoring"** → leave all as default.

Click **"Next: Advanced"** → leave all as default.

### 7. Review and Create

Click **"Review + Create"**. Wait for validation to pass, then click **"Create"**.

> If you selected SSH public key, download the `.pem` key file when prompted — you will need it to SSH into the VM.

### 8. Wait for Deployment to Complete

The deployment typically takes 1–3 minutes. Once complete, click **"Go to resource"**.

### 9. Get the VM's Public IP Address

On the VM overview page, note the **Public IP address** displayed on the right side.

### 10. SSH into the VM

Open a terminal on your client host and run:

**If using SSH key:**
```bash
chmod 400 <downloaded-key>.pem
ssh -i <downloaded-key>.pem azureuser@<PublicIPAddress>
```

**If using password:**
```bash
ssh azureuser@<PublicIPAddress>
```

---

## Expected Output

**Successful SSH login:**
```
Welcome to Ubuntu 24.04 LTS (GNU/Linux 6.x.x-azure x86_64)
azureuser@devops-vm:~$
```

---

## VM Configuration Summary

| Property | Value |
|----------|-------|
| Name | `devops-vm` |
| Region | `East US` |
| Image | Ubuntu Server 24.04 LTS |
| Size | Standard_B1s (1 vCPU, 1 GiB RAM) |
| OS Disk Type | Standard HDD |
| OS Disk Size | 30 GiB |
| Inbound Ports | SSH (22) |
| Authentication | SSH key or Password |

---

## Key Concepts

- **Virtual Machine (VM):** A software-based computer that runs on physical hardware in an Azure data center. VMs give full control over the operating system and installed software.
- **Ubuntu 24.04 LTS:** A Long Term Support release of Ubuntu Linux, supported for 5 years. LTS releases are preferred for production and lab environments due to their stability.
- **Standard_B1s:** A burstable VM size with 1 vCPU and 1 GiB RAM. Suitable for lightweight workloads and dev/test environments. The "B" series accumulates CPU credits during idle periods and spends them during bursts.
- **Standard HDD:** The most cost-effective disk type, using magnetic storage. Best for dev/test workloads and backups where high IOPS are not required.
- **NSG (Network Security Group):** A firewall that filters inbound and outbound traffic. Selecting "Basic" NSG with SSH (22) automatically creates an allow rule for port 22.
- **azureuser:** The default administrative user created on Azure Linux VMs. It has sudo privileges.

---

## Notes

- If you generated an SSH key pair during VM creation, download and store the `.pem` file securely — Azure does not store the private key and it cannot be retrieved later.
- The VM's public IP address is **dynamic** by default and may change if the VM is stopped and restarted. Use a static IP if a persistent address is needed.
- Always deallocate VMs when not in use in lab environments to avoid consuming credits.
- To verify the VM is running from the CLI:
  ```bash
  az vm show \
    --name devops-vm \
    --resource-group <ResourceGroupName> \
    --show-details \
    --query "{Name:name, State:powerState, PublicIP:publicIps}" \
    -o table
  ```

---

