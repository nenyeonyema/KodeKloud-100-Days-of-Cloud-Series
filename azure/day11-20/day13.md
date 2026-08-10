# Task 13 - Configure Passwordless SSH Access to an Azure VM

## Overview

This task is part of the Nautilus DevOps team's incremental cloud migration strategy to Microsoft Azure. Secure and password-less SSH access is a fundamental requirement for managing virtual machines in a DevOps environment. This task focuses on copying the root user's SSH public key from the Azure client host to the `xfusion-vm` Azure VM, enabling secure password-less SSH access as the root user.

---

## Objectives

- Copy the SSH public key of the root user from the Azure client host located at `/root/.ssh/id_rsa.pub` to the `authorized_keys` file of the root user on `xfusion-vm`
- Ensure proper permissions are set on the `.ssh` folder and `authorized_keys` file on the VM
- Verify that passwordless SSH access works from the Azure client host to `xfusion-vm` as the root user

---

## Prerequisites

- Access to an active Azure subscription or lab credentials
- Azure CLI installed and authenticated
- An existing Resource Group
- An existing VM named `xfusion-vm` running in the `southcentralus` region
- The default SSH user on the VM is `azureuser`
- SSH public key of the root user available at `/root/.ssh/id_rsa.pub` on the Azure client host
- Port 22 open on the VM's Network Security Group (NSG)

---

## Tools Used

| Tool | Purpose |
|------|---------|
| Azure CLI | Managing VM and NSG rules |
| ssh-copy-id | Copying SSH public key to the VM |
| SSH | Connecting to the VM |
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

### 3. Get the VM's Public IP Address

```bash
az vm show \
  --name xfusion-vm \
  --resource-group <ResourceGroupName> \
  --show-details \
  --query "publicIps" \
  -o tsv
```

### 4. Ensure Port 22 is Open on the NSG

```bash
az vm open-port \
  --name xfusion-vm \
  --resource-group <ResourceGroupName> \
  --port 22
```

### 5. View the Root User's SSH Public Key

```bash
cat /root/.ssh/id_rsa.pub
```

### 6. Copy the Public Key to the VM as azureuser

```bash
ssh-copy-id -i /root/.ssh/id_rsa.pub azureuser@<PublicIPFromStep3>
```

### 7. SSH into the VM as azureuser

```bash
ssh azureuser@<PublicIPFromStep3>
```

### 8. Add the Public Key to Root's authorized_keys

Once inside the VM, run the following commands:

```bash
sudo mkdir -p /root/.ssh
sudo chmod 700 /root/.ssh
sudo cp /home/azureuser/.ssh/authorized_keys /root/.ssh/authorized_keys
sudo chmod 600 /root/.ssh/authorized_keys
```

### 9. Exit Back to the Azure Client Host

```bash
exit
```

### 10. Verify Passwordless SSH as Root

```bash
ssh root@<PublicIPFromStep3>
```

If you are logged in without being prompted for a password, the task is complete.

---

## Expected Output

**Step 6 - ssh-copy-id:**
```
/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/root/.ssh/id_rsa.pub"
Number of key(s) added: 1
Now try logging into the machine with: ssh 'azureuser@<PublicIP>'
```

**Step 10 - Passwordless SSH as root:**
```
Welcome to Ubuntu ...
root@xfusion-vm:~#
```

---

## File Permission Reference

| Path | Owner | Permission | Octal |
|------|-------|------------|-------|
| `/root/.ssh` | root | drwx------ | 700 |
| `/root/.ssh/authorized_keys` | root | -rw------- | 600 |

> Incorrect permissions on `.ssh` or `authorized_keys` will cause SSH to silently reject the key and fall back to password authentication.

---

## Key Concepts

- **SSH Key-Based Authentication:** A more secure alternative to password authentication. Uses a public-private key pair where the public key is stored on the server and the private key remains on the client.
- **authorized_keys:** A file on the SSH server that contains a list of public keys permitted to log in. When a client connects, the server checks if the client's public key matches any entry in this file.
- **ssh-copy-id:** A utility that installs a public key on a remote server's `authorized_keys` file automatically and safely.
- **azureuser:** The default administrative user created on Azure Linux VMs. It has sudo privileges but is not the root user.
- **sudo:** Allows `azureuser` to run commands as root, which is necessary for modifying `/root/.ssh/`.

---

## Notes

- Direct root SSH login may be disabled by default on Azure VMs (`PermitRootLogin` in `/etc/ssh/sshd_config`). Copying the key via `azureuser` and then propagating it to root's `authorized_keys` is the recommended approach.
- Always set correct permissions on `.ssh` (700) and `authorized_keys` (600) — SSH will silently reject keys if permissions are too open.
- The `sudo cp` command copies the `azureuser`'s `authorized_keys` to root's `.ssh` directory since both use the same key in this scenario.
- After completing this task, the root user on the VM can be accessed directly from the Azure client host without a password.

---

