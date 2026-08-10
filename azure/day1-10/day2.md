# Azure Task 2: Create and Access an Azure Virtual Machine

## Overview

In this task, I created an Azure Virtual Machine according to the specified requirements and verified that I could connect to it using SSH.

The VM was configured with:

| Configuration | Value |
|---|---|
| VM Name | `devops-vm` |
| Region | `East US` |
| Image | Ubuntu Server 24.04 LTS |
| VM Size | `Standard_B1s` |
| OS Disk | Default |
| Data Disk | 30 GB Standard HDD |
| Network Security Group | Default NSG |
| Inbound Access | SSH (TCP 22) |

---

## Prerequisites

- Azure subscription
- Azure CLI
- Access to the Azure client/landing host
- Azure lab credentials
- SSH client

---

* 1 Retrieve the Lab Credentials

On the Azure client host, retrieve the temporary Azure credentials:

```bash
showcreds
```

* 2 Log in to Azure
Authenticate with Azure CLI:

```bash
az login
```

Verify the active subscription:

```bash
az account show
```
* 3 Find the Existing Resource Group
The task requires using the existing resource group.

List the resource groups:

```
az group list \
  --query "[].name" \
  --output table
```  
If the lab resource group follows the kml naming convention, it can be located with:
```bash
az group list \
  --query "[].name" \
  --output table | grep "kml"
```
Store the resource group name for the following commands:

RG="<resource-group>"

For example:

```bash
RG="kml_rg_main-xxxxxxxx"
```
* 4 Check Available Ubuntu 24.04 Images
Before creating the VM, search for the Ubuntu 24.04 LTS image:

```bash
az vm image list \
  --location eastus \
  --publisher Canonical \
  --offer ubuntu-24_04-lts \
  --all \
  --output table
```
The VM should use the Ubuntu 24.04 LTS image.

* 5 Create the Virtual Machine
Create the VM with the required configuration:

```bash
az vm create \
  --resource-group "$RG" \
  --name devops-vm \
  --location eastus \
  --image Canonical:ubuntu-24_04-lts:server:24.04.202410020 \
  --size Standard_B1s \
  --admin-username azureuser \
  --generate-ssh-keys \
  --storage-sku Standard_LRS \
  --os-disk-size-gb 30 \
  --public-ip-sku Standard
```
The exact Ubuntu image URN available in the lab may differ. If the image URN returned by az vm image list is different, use the available Ubuntu 24.04 LTS image from that result.

* 6 Configure SSH Access
Allow inbound SSH traffic on port 22:

```
az vm open-port \
  --resource-group "$RG" \
  --name devops-vm \
  --port 22 \
  --priority 1000
```

This creates or updates the network security configuration to allow SSH traffic.

* 7 Add the 30 GB Standard HDD Data Disk
Create a 30 GB managed disk using Standard HDD:

```bash
az disk create \
  --resource-group "$RG" \
  --name devops-vm-data-disk \
  --size-gb 30 \
  --sku Standard_LRS \
  --location eastus
```
Attach the Disk of the VM:

```
az vm disk attach \
  --resource-group "$RG" \
  --vm-name devops-vm \
  --name devops-vm-data-disk
```

* 8 Verify the VM Configuration

Check the VM details:

```bash
az vm show \
  --resource-group "$RG" \
  --name devops-vm \
  --show-details \
  --output table
```

Verify the VM size:

```bash
az vm show \
  --resource-group "$RG" \
  --name devops-vm \
  --query "hardwareProfile.vmSize"
```

Expected:

```bash
Standard_B1s"
```
Verify the VM location:

```bash
az vm show \
  --resource-group "$RG" \
  --name devops-vm \
  --query "location"
```
Expected:

`eastus`

* 9 Verify the Attached Data Disk
List the disks attached to the VM:

```bash
az vm show \
  --resource-group "$RG" \
  --name devops-vm \
  --query "storageProfile.dataDisks[].{Name:name,SizeGB:diskSizeGb,LUN:lun}"
Expected result should include the 30 GB data disk:
```
```bash
Name                  SizeGB
--------------------  ------
devops-vm-data-disk   30
Verify the disk SKU:
```

```
az disk show \
  --resource-group "$RG" \
  --name devops-vm-data-disk \
  --query "{name:name,sizeGB:diskSizeGb,sku:sku.name}"
Expected:

{
  "name": "devops-vm-data-disk",
  "sizeGB": 30,
  "sku": "Standard_LRS"
}
```

* 10 Get the VM Public IP
Retrieve the public IP address:

```bash
az vm show \
  --resource-group "$RG" \
  --name devops-vm \
  --show-details \
  --query "publicIps" \
  --output tsv
Store the IP address:

VM_IP=$(az vm show \
  --resource-group "$RG" \
  --name devops-vm \
  --show-details \
  --query "publicIps" \
  --output tsv)
```
** Verify: **

echo `$VM_IP`

* 11 Verify SSH Port 22
Check that the NSG allows SSH:

```
az network nsg rule list \
  --resource-group "$RG" \
  --nsg-name "$(az vm show \
    --resource-group "$RG" \
    --name devops-vm \
    --query "networkProfile.networkInterfaces[0].id" \
    --output tsv | xargs -n1 basename | xargs az network nic show \
    --resource-group "$RG" \
    --query "networkSecurityGroup.id" \
    --output tsv | xargs -n1 basename)" \
  --output table
```
Alternatively, inspect the VM's NIC and NSG through the Azure Portal.

The required inbound rule is:
```bash
Protocol: TCP
Port:     22
Access:   Allow
Step 12: SSH into the VM
Use the azureuser account specified by the task:
```
```bash
ssh azureuser@"$VM_IP"
```
If the SSH key was generated automatically by Azure CLI, the corresponding private key should be available under:

```
~/.ssh/
```
If prompted about the host authenticity, confirm with:

yes

* 12	 Verify the VM from Inside the Server
After connecting:

hostname
Expected:

devops-vm
Verify the operating system:

```
cat /etc/os-release
```
The output should show Ubuntu 24.04 LTS.

