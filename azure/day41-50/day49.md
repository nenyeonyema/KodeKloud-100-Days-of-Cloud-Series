# Azure Task 49 — Static Web App Served by Nginx from Azure Blob Storage

## Overview

This task demonstrates a secure static content delivery pattern where an index.html file is stored in a private Azure Blob Storage container and downloaded to an Nginx web server running on a VM. The Storage Account is kept private — no public blob access, no static website hosting feature. The VM fetches the file directly from Blob Storage using the Azure CLI with account key authentication, then serves it locally via Nginx. This pattern separates static assets from the main source code repository while keeping storage private and secure.

---

## Architecture

```
+------------------------------------------+
|         xfusionstor2514                  |
|         Storage Account (Private)        |
|         East US | Standard_LRS           |
|         Public blob access: DISABLED     |
|                                          |
|   +------------------------------------+ |
|   |      xfusion-container             | |
|   |      index.html                    | |
|   +------------------------------------+ |
+--------------------+---------------------+
                     |
                     | az storage blob download
                     | (account key auth)
                     v
+------------------------------------------+
|         xfusion-vnet (10.0.0.0/16)       |
|                                          |
|   +------------------------------------+ |
|   |      xfusion-subnet (10.0.1.0/24)  | |
|   |                                    | |
|   |      xfusion-vm                    | |
|   |      East US | Ubuntu 22.04        | |
|   |      20.102.71.7                   | |
|   |                                    | |
|   |      Nginx serving:                | |
|   |      /var/www/html/index.html      | |
|   +------------------------------------+ |
+------------------------------------------+
                     |
                     | HTTP :80
                     v
              curl http://20.102.71.7
              Welcome to KKE Azure Labs!
```

---

## Prerequisites

- Azure CLI installed and authenticated
- Resource group: kml_rg_main-22a1cf356f514929
- /root/index.html exists on lab host
- Region: East US

---

## Step 1 — Login and Get Resource Group

```bash
az login -u kk_lab_user_main-22a1cf356f514929@azurefreekmlprod.onmicrosoft.com -p "kVk8%Yfg"
az group list --query "[0].name" -o tsv
```

Output: kml_rg_main-22a1cf356f514929

---

## Step 2 — Create VNet and Subnet

```bash
az network vnet create \
  --resource-group kml_rg_main-22a1cf356f514929 \
  --name xfusion-vnet \
  --location eastus \
  --address-prefix 10.0.0.0/16 \
  --subnet-name xfusion-subnet \
  --subnet-prefix 10.0.1.0/24
```

---

## Step 3 — Create Private Storage Account

```bash
az storage account create \
  --name xfusionstor2514 \
  --resource-group kml_rg_main-22a1cf356f514929 \
  --location eastus \
  --sku Standard_LRS \
  --allow-blob-public-access false \
  --public-network-access Enabled
```

**Flags explained:**
- `--allow-blob-public-access false` — disables anonymous public access to all blobs
- `--public-network-access Enabled` — keeps the storage reachable from the VM via authenticated requests; the VM needs to download from it

---

## Step 4 — Create Blob Container and Upload index.html

```bash
az storage container create \
  --name xfusion-container \
  --account-name xfusionstor2514 \
  --auth-mode login

az storage blob upload \
  --account-name xfusionstor2514 \
  --container-name xfusion-container \
  --name index.html \
  --file /root/index.html \
  --auth-mode login
```

The index.html content:
```html
<!DOCTYPE html>
<html lang=en>
<head>
    <meta charset=UTF-8>
    <meta name=viewport content=width=device-width, initial-scale=1.0>
    <title>Welcome</title>
</head>
<body>
    <h1>Welcome to KKE Azure Labs!</h1>
</body>
</html>
```

---

## Step 5 — Get Storage Account Key

```bash
az storage account keys list \
  --account-name xfusionstor2514 \
  --resource-group kml_rg_main-22a1cf356f514929 \
  --query "[0].value" -o tsv
```

---

## Step 6 — Generate SSH Key

```bash
ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa -N ""
```

---

## Step 7 — Create VM with User Data

```bash
cat > /tmp/userdata.sh << 'EOF'
#!/bin/bash
apt-get update -y
apt-get install -y nginx azure-cli
systemctl start nginx
systemctl enable nginx
EOF

az vm create \
  --resource-group kml_rg_main-22a1cf356f514929 \
  --name xfusion-vm \
  --image Ubuntu2204 \
  --size Standard_B1s \
  --admin-username azureuser \
  --ssh-key-values ~/.ssh/id_rsa.pub \
  --vnet-name xfusion-vnet \
  --subnet xfusion-subnet \
  --public-ip-sku Standard \
  --location eastus \
  --storage-sku Standard_LRS \
  --os-disk-size-gb 30 \
  --nsg-rule SSH \
  --custom-data /tmp/userdata.sh
```

VM output:
- Public IP: 20.102.71.7
- Private IP: 10.0.1.4
- State: VM running

---

## Step 8 — Open Port 80

```bash
az vm open-port \
  --resource-group kml_rg_main-22a1cf356f514929 \
  --name xfusion-vm \
  --port 80
```

---

## Step 9 — Install Nginx and Azure CLI on VM

The user data script may not have completed by the time SSH is available. Install manually to be sure:

```bash
ssh azureuser@20.102.71.7
```

Once inside:
```bash
sudo apt-get update -y
sudo apt-get install -y nginx
sudo systemctl start nginx
sudo systemctl enable nginx
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
```

**Note:** `sudo az` returned `command not found` when running remotely via SSH one-liner — installing interactively inside the SSH session resolved this.

---

## Step 10 — Download index.html from Blob to Nginx Web Root

Still inside the VM:
```bash
sudo az storage blob download \
  --account-name xfusionstor2514 \
  --account-key '<STORAGE_KEY>' \
  --container-name xfusion-container \
  --name index.html \
  --file /var/www/html/index.html

cat /var/www/html/index.html
exit
```

---

## Step 11 — Verify

```bash
curl http://20.102.71.7
```

Output:
```html
<!DOCTYPE html>
<html lang=en>
<head>
    <meta charset=UTF-8>
    <meta name=viewport content=width=device-width, initial-scale=1.0>
    <title>Welcome</title>
</head>
<body>
    <h1>Welcome to KKE Azure Labs!</h1>
</body>
</html>
```

---

## Key Concepts

**Why keep Blob Storage private?**
The task intentionally keeps the storage account private (`--allow-blob-public-access false`). The index.html is separated from the main source code repository to avoid exposing internal application code. Making the blob public would allow anyone with the URL to access it — using account key authentication keeps access controlled and auditable.

**Why not use Static Website hosting?**
Azure Storage's Static Website feature serves files directly from Blob Storage via a public URL. This task deliberately avoids it because the goal is for the VM (Nginx) to serve the content locally after downloading it — giving the team control over caching, headers, routing, and future dynamic content without changing the serving infrastructure.

**Why not mount the storage account?**
Mounting Azure Blob Storage (via blobfuse or NFS) adds complexity and a persistent dependency on storage availability. The download approach is simpler — the file is fetched once and served locally by Nginx. If the blob is updated, the VM can re-fetch it on demand.

**The content delivery flow**

```
Developer pushes index.html to Blob Storage
         |
         | az storage blob upload
         v
xfusion-container (private)
         |
         | az storage blob download (account key)
         v
/var/www/html/index.html on xfusion-vm
         |
         | Nginx serves locally
         v
Browser: http://20.102.71.7
```

**Why `--public-network-access Enabled` with `--allow-blob-public-access false`?**
These are two different settings:

| Setting | What it controls |
|---|---|
| `--allow-blob-public-access false` | Blocks anonymous/public reads of blobs |
| `--public-network-access Enabled` | Allows authenticated requests from public internet |

Disabling public network access entirely would block the VM from downloading the blob unless a private endpoint was configured. Since the VM uses account key auth over the public internet, public network access must remain enabled while anonymous blob access is disabled.

---

## Lessons Learned

1. User data scripts run asynchronously after VM creation — if the VM boots faster than the script completes, tools installed by user data (like Azure CLI) won't be available immediately. Installing interactively via SSH is more reliable for time-sensitive tasks.
2. `sudo az` via SSH one-liner returns `command not found` even when az is installed — this is because sudo uses a restricted PATH that doesn't include `/usr/bin/az`. Fix with `sudo env "PATH=$PATH" az ...` or run interactively inside the SSH session.
3. Always use `--allow-blob-public-access false` for private storage — this is now the recommended default for all production storage accounts.
4. `--storage-sku Standard_LRS` and `--os-disk-size-gb 30` are required for VMs in policy-restricted lab subscriptions to avoid Premium SSD policy violations.

---

## Outcome

| Task | Result |
|---|---|
| xfusion-vnet created (10.0.0.0/16) | Done |
| xfusion-subnet created (10.0.1.0/24) | Done |
| xfusionstor2514 created (Standard_LRS, private) | Done |
| xfusion-container created | Done |
| index.html uploaded to blob container | Done |
| xfusion-vm created (East US, B1s, Standard_LRS) | Done |
| Port 80 opened on NSG | Done |
| Nginx installed and running | Done |
| Azure CLI installed on VM | Done |
| index.html downloaded from blob to /var/www/html/ | Done |
| curl http://20.102.71.7 → Welcome to KKE Azure Labs! | Done |
