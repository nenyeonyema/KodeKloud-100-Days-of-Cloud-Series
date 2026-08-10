# Azure Task 1 — Create an SSH Key Pair

## Overview

In this task, an SSH key pair was created in Microsoft Azure for secure authentication to Azure Virtual Machines.

The key pair was created with the following configuration:

| Configuration | Value |
|---|---|
| SSH Key Name | `datacenter-kp` |
| Key Type | `RSA` |

## Objective

Create an RSA SSH key pair named `datacenter-kp` that can be used for SSH authentication to Azure Virtual Machines.

## Prerequisites

- Azure account with access to the target subscription
- Azure Portal or Azure CLI
- Azure resource group
- SSH client

---

## Method 1: Azure Portal

1. Log in to the [Azure Portal](https://portal.azure.com).

2. Search for **SSH keys**.

3. Select **SSH keys** and click **Create**.

4. Configure the SSH key:

   - **Subscription:** Select the lab subscription
   - **Resource Group:** Select the existing resource group
   - **Region:** Select the required region
   - **SSH Key Name:** `datacenter-kp`
   - **SSH Key Type:** `RSA`
   - **SSH Public Key Source:** `Generate new key pair`

5. Click **Review + Create**.

6. Click **Create**.

7. When prompted, download the generated private key.

The private key should be stored securely and should never be committed to a Git repository.

---

## Method 2: Azure CLI

The Azure CLI can also be used to generate an RSA key locally and upload the public key to Azure.

### 1. Generate the RSA Key Pair

```bash
ssh-keygen -t rsa -b 2048 -f ~/.ssh/datacenter-kp
```

This creates two files:
```bash
~/.ssh/datacenter-kp
~/.ssh/datacenter-kp.pub
```

Where:

* datacenter-kp — private key

* datacenter-kp.pub — public key

2. Log in to Azure
```bash
az login
```

3. Create the Azure SSH Key Resource
Replace <resource-group> and <location> with the appropriate lab values:
```bash
az sshkey create \
  --resource-group <resource-group> \
  --name datacenter-kp \
  --location <location> \
  --public-key ~/.ssh/datacenter-kp.pub
```

4. Verify the SSH Key
```
az sshkey show \
  --resource-group <resource-group> \
  --name datacenter-kp
```
**SSH Key Permissions**
The private key should have restrictive permissions:

```bash
chmod 600 ~/.ssh/datacenter-kp
```

**Verify the permissions:**

```bash
ls -l ~/.ssh/datacenter-kp
```

