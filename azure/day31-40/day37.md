# Azure Task 37 — PHP Application to MySQL Database Integration

## Overview

This task demonstrates integrating a PHP application hosted on one Azure VM with a MySQL database hosted on a separate Azure VM. The goal is to validate cross-VM database connectivity in the cloud using a Percona MySQL server and a pre-existing PHP application.

---

## Architecture

```
                    Port 22 (SSH)
                   ┌─────────────┐
                   │  Lab Host   │
                   └──────┬──────┘
                          │ SSH
          ┌───────────────┼───────────────┐
          ▼                               ▼
┌─────────────────────┐       ┌─────────────────────┐
│   devops-mysql-vm   │       │   devops-php-vm      │
│   Central US        │◄──────│   East US            │
│   10.0.0.4          │ :3306 │                      │
│   20.9.53.74 (pub)  │       │   /var/www/html/     │
│                     │       │   db_test.php        │
│   Percona MySQL 5.7 │       │   Apache + PHP       │
└─────────────────────┘       └─────────────────────┘
         ▲
         │ NSG Inbound Rule
         │ Port 3306 (MySQL)
```

---

## Prerequisites

- Azure CLI installed and authenticated
- Resource group: `kml_rg_main-51f30c541977493c`
- `devops-php-vm` already exists in East US with passwordless SSH from lab host
- Percona MySQL image available on Azure Marketplace: `jetware-srl:percona_mysql:percona_mysql57-ubuntu-1604:1.0.170503`

---

## Phase 1 — Create the MySQL VM

### Login
```bash
az login -u kk_lab_user_main-51f30c541977493c@azurefreekmlprod.onmicrosoft.com -p "v8kBXHan"
```

### Create the VM
```bash
az vm create \
  --resource-group kml_rg_main-51f30c541977493c \
  --name devops-mysql-vm \
  --image jetware-srl:percona_mysql:percona_mysql57-ubuntu-1604:1.0.170503 \
  --plan-name percona_mysql57-ubuntu-1604 \
  --plan-publisher jetware-srl \
  --plan-product percona_mysql \
  --location centralus \
  --size Standard_B1s \
  --admin-username devops_admin \
  --admin-password "Namin@123456" \
  --authentication-type password \
  --storage-sku Standard_LRS \
  --public-ip-sku Standard \
  --nsg-rule SSH
```

**Notes:**
- `--storage-sku Standard_LRS` sets the OS disk to Standard HDD
- `--nsg-rule SSH` opens port 22 at creation time
- `--plan-*` flags are required for Marketplace images with billing plans

### VM Output
```json
{
  "location": "centralus",
  "powerState": "VM running",
  "privateIpAddress": "10.0.0.4",
  "publicIpAddress": "20.9.53.74"
}
```

---

## Phase 2 — Add Port 3306 NSG Rule

Port 3306 cannot be selected during VM creation — it must be added post-creation via the NSG.

### Get NSG name
```bash
az network nsg list \
  --resource-group kml_rg_main-51f30c541977493c \
  --query "[?contains(name,'mysql')].name" -o tsv
```

### Add MySQL inbound rule
```bash
az network nsg rule create \
  --resource-group kml_rg_main-51f30c541977493c \
  --nsg-name devops-mysql-vmNSG \
  --name Allow-MySQL \
  --priority 110 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --destination-port-ranges 3306 \
  --source-address-prefixes '*' \
  --destination-address-prefixes '*'
```

---

## Phase 3 — Set Up MySQL Database

### SSH into the MySQL VM
```bash
ssh devops_admin@20.9.53.74
```

### Access the MySQL shell via Jetware wrapper
```bash
sudo /jet/enter mysql
```

### Create database, user, and grant privileges
```sql
CREATE DATABASE devops_db;
CREATE USER 'devops_user'@'%' IDENTIFIED BY 'password123';
GRANT ALL PRIVILEGES ON devops_db.* TO 'devops_user'@'%';
FLUSH PRIVILEGES;
EXIT;
```

**Why `'%'` as the host?**
The `%` wildcard allows `devops_user` to connect from any IP address, which is required for the PHP VM to connect remotely. Without it, MySQL would only accept connections from localhost.

---

## Phase 4 — Configure the PHP Application

### Get the PHP VM's public IP
```bash
az vm list-ip-addresses \
  --resource-group kml_rg_main-51f30c541977493c \
  --name devops-php-vm \
  --query "[].virtualMachine.network.publicIpAddresses[0].ipAddress" -o tsv
```

### SSH into the PHP VM (passwordless)
```bash
ssh azureuser@<PHP_VM_IP>
```

### Edit the PHP connection file
```bash
sudo nano /var/www/html/db_test.php
```

Update the connection variables:
```php
$servername = "20.9.53.74";
$username = "devops_user";
$password = "password123";
$dbname = "devops_db";
$port = 3306;
$conn = new mysqli($servername, $username, $password, $dbname, $port);
```

### Verify PHP syntax
```bash
php -l /var/www/html/db_test.php
```

Expected output:
```
No syntax errors detected in /var/www/html/db_test.php
```

---

## Phase 5 — Validation

Access the PHP file in a browser:
```
http://<PHP_VM_IP>/db_test.php
```

Expected output:
```
Connected successfully
```

---

## Key Concepts

**Why Percona Server for MySQL?**
Percona is a drop-in replacement for MySQL with performance and observability improvements. The Jetware image bundles it with a pre-configured environment accessible via `sudo /jet/enter mysql`.

**Why `--storage-sku Standard_LRS` instead of `--os-disk-sku`?**
The `--os-disk-sku` flag is not recognized in this version of the Azure CLI. The correct flag for setting disk type at VM creation is `--storage-sku`.

**Why add port 3306 after VM creation?**
The Azure VM creation wizard only exposes SSH (22), RDP (3389), and HTTP (80) as selectable inbound ports. MySQL's port 3306 must be added as a custom NSG rule post-creation.

**Why use the port as the fifth argument in mysqli?**
The default `new mysqli($host, $user, $pass, $db)` assumes port 3306. Explicitly passing it as the fifth argument makes the configuration portable and avoids relying on MySQL client defaults.

---

## Outcome

| Task | Result |
|---|---|
| devops-mysql-vm created (Central US, B1s, Standard HDD) | ✅ |
| Port 3306 NSG inbound rule added | ✅ |
| devops_db database created | ✅ |
| devops_user@'%' created with password123 | ✅ |
| All privileges granted and flushed | ✅ |
| db_test.php updated with correct connection variables | ✅ |
| PHP syntax check passed | ✅ |
| Browser validation → Connected successfully | ✅ |
