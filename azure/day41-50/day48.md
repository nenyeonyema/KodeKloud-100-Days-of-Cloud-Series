# Azure Task 48 — ACR, Docker, Blob Storage and VM Integration

## Overview

This task demonstrates a full containerized application deployment on Azure. A Python Flask app is containerized using Docker, pushed to Azure Container Registry (ACR), deployed on a VM, and configured to read a config file from Azure Blob Storage. The task covers ACR setup, local Docker build and push, VM provisioning, Docker installation, and runtime config injection.

---

## Architecture

```
+-------------------+
|    Lab Host       |
|    /root/pyapp/   |
|    Dockerfile     |
|    app.py         |
|    config.json    |
+--------+----------+
         |
         | docker build + push
         v
+----------------------------------+
|   datacenteracr30062             |
|   Azure Container Registry       |
|   East US | Basic SKU            |
|                                  |
|   datacenter/python-app:latest   |
+----------------+-----------------+
                 |
                 | docker pull + run
                 v
+----------------------------------+      +----------------------------------+
|   datacenter-vm                  |      |   datacenterstor30062            |
|   East US | Ubuntu 22.04         |      |   Storage Account | Standard_LRS |
|   20.124.0.57                    |      |                                  |
|                                  |      |   +----------------------------+ |
|   Docker container:              |      |   |   datacenter-config        | |
|   python-app                     |      |   |   config.json              | |
|   port 80                        |      |   +----------------------------+ |
|                                  |      +----------------------------------+
|   /app/config.json (injected)    |
+----------------------------------+
         |
         | HTTP :80
         v
   curl http://20.124.0.57
   Welcome to KKE Azure Labs: {'key': 'value', 'version': 1}
```

---

## Prerequisites

- Azure CLI installed and authenticated
- Docker installed on lab host
- Resource group: kml_rg_main-af17cbca7f8c49a0
- /root/pyapp/Dockerfile and /root/pyapp/app.py exist
- /root/config.json exists
- Region: East US

---

## Step 1 — Login and Get Resource Group

```bash
az login -u kk_lab_user_main-af17cbca7f8c49a0@azurefreekmlprod.onmicrosoft.com -p "f#98zZfJ"
az group list --query "[0].name" -o tsv
```

Output: kml_rg_main-af17cbca7f8c49a0

---

## Step 2 — Create Azure Container Registry

```bash
az acr create \
  --resource-group kml_rg_main-af17cbca7f8c49a0 \
  --name datacenteracr30062 \
  --sku Basic \
  --location eastus
```

ACR login server: `datacenteracr30062.azurecr.io`

---

## Step 3 — Enable Admin and Get ACR Credentials

`az acr build` was blocked by subscription policy (no `scheduleRun` permission). The workaround is to build locally with Docker and push using admin credentials:

```bash
az acr update \
  --name datacenteracr30062 \
  --admin-enabled true

az acr credential show \
  --name datacenteracr30062 \
  --query "{username:username, password:passwords[0].value}" \
  -o json
```

---

## Step 4 — Build and Push Docker Image

```bash
# Login to ACR
docker login datacenteracr30062.azurecr.io \
  --username datacenteracr30062 \
  --password <ACR_PASSWORD>

# Build image
docker build -t datacenteracr30062.azurecr.io/datacenter/python-app:latest /root/pyapp

# Push to ACR
docker push datacenteracr30062.azurecr.io/datacenter/python-app:latest
```

Push confirmed:
```
latest: digest: sha256:afdc8cbe2d80bf825f59788a0f188fc7e08d97534d5f1e0667232c82d3be658f size: 1783
```

### The Dockerfile

```dockerfile
FROM python:3.9-slim
WORKDIR /app
COPY app.py /app/
RUN pip install flask
CMD ["python", "app.py"]
```

### The Application (app.py)

```python
from flask import Flask
import json

app = Flask(__name__)

@app.route("/")
def home():
    with open("config.json", "r") as f:
        config = json.load(f)
    return f"Welcome to KKE Azure Labs: {config}"

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=80)
```

---

## Step 5 — Create Storage Account and Upload config.json

```bash
az storage account create \
  --name datacenterstor30062 \
  --resource-group kml_rg_main-af17cbca7f8c49a0 \
  --location eastus \
  --sku Standard_LRS

az storage container create \
  --name datacenter-config \
  --account-name datacenterstor30062 \
  --auth-mode login

az storage blob upload \
  --account-name datacenterstor30062 \
  --container-name datacenter-config \
  --name config.json \
  --file /root/config.json \
  --auth-mode login
```

config.json contents:
```json
{
  "key": "value",
  "version": 1
}
```

---

## Step 6 — Create VM

```bash
ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa -N ""

az vm create \
  --resource-group kml_rg_main-af17cbca7f8c49a0 \
  --name datacenter-vm \
  --image Ubuntu2204 \
  --size Standard_B1s \
  --admin-username azureuser \
  --ssh-key-values ~/.ssh/id_rsa.pub \
  --public-ip-sku Standard \
  --location eastus \
  --storage-sku Standard_LRS \
  --os-disk-size-gb 30 \
  --nsg-rule SSH

az vm open-port \
  --resource-group kml_rg_main-af17cbca7f8c49a0 \
  --name datacenter-vm \
  --port 80
```

VM output:
- Public IP: 20.124.0.57
- Private IP: 10.0.0.4
- State: VM running

---

## Step 7 — Install Docker and Azure CLI on VM

```bash
ssh azureuser@20.124.0.57 << 'ENDSSH'
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker azureuser
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
ENDSSH
```

---

## Step 8 — Pull and Run Container on VM

```bash
ssh azureuser@20.124.0.57 << 'ENDSSH'
sudo docker login datacenteracr30062.azurecr.io \
  --username datacenteracr30062 \
  --password <ACR_PASSWORD>

sudo docker pull datacenteracr30062.azurecr.io/datacenter/python-app:latest
sudo docker run -d -p 80:80 --name python-app datacenteracr30062.azurecr.io/datacenter/python-app:latest
sudo docker ps
ENDSSH
```

---

## Step 9 — Inject config.json into Running Container

The Dockerfile only copies `app.py` into the image — `config.json` is not included. The app reads it at runtime from `/app/config.json`. Without it, Flask returns a 500 error.

Fix — copy config.json from lab host into the running container:

```bash
# Copy config.json to VM
scp /root/config.json azureuser@20.124.0.57:/tmp/config.json

# Copy from VM into running container
ssh azureuser@20.124.0.57 "sudo docker cp /tmp/config.json python-app:/app/config.json"
```
[O
---

## Step 10 — Verify Application

```bash
curl http://20.124.0.57
```

Output:
```
Welcome to KKE Azure Labs: {'key': 'value', 'version': 1}
```

---

## Key Concepts

**Why `az acr build` failed**
`az acr build` runs the build on Azure's infrastructure using ACR Tasks. It requires the `scheduleRun` permission which was blocked by the lab subscription policy. The workaround is building locally with `docker build` and pushing with `docker push` using ACR admin credentials.

**Why the app returned 500 on first run**
The Dockerfile copies only `app.py` into the image:
```dockerfile
COPY app.py /app/
```
The `config.json` file is not included. When Flask calls `open("config.json", "r")`, the file doesn't exist in `/app/` inside the container — causing a FileNotFoundError and a 500 response. The fix is to inject the file at runtime using `docker cp`.

**Better long-term solutions for config injection**

| Method | How |
|---|---|
| Docker volume mount | `docker run -v /path/config.json:/app/config.json` |
| Environment variables | Pass config values as `-e KEY=value` |
| Fetch from Blob at startup | App downloads config.json from Azure Blob on boot |
| Update Dockerfile | `COPY config.json /app/` in the build |

For this task, `docker cp` was the fastest fix since the container was already running.

**ACR vs Docker Hub**

| Feature | ACR | Docker Hub |
|---|---|---|
| Privacy | Private by default | Public by default |
| Integration | Native Azure RBAC, AKS, VMs | Manual credential setup |
| Location | Same Azure region = faster pulls | Global CDN |
| Cost | Included in Basic SKU | Free tier available |

**Why `--os-disk-size-gb 30` and `--storage-sku Standard_LRS`**
The lab subscription policy blocks Premium SSD disks and disks larger than 128GB. Ubuntu 22.04 defaults to Premium SSD — explicitly setting Standard_LRS and a small disk size satisfies the policy.

---

## Lessons Learned

1. `az acr build` requires `scheduleRun` permission — use local `docker build` + `docker push` as a fallback.
2. Always check what the Dockerfile copies — missing runtime files cause 500 errors, not build failures.
3. `docker cp` is the fastest way to inject files into a running container without rebuilding the image.
4. Always open port 80 on the VM NSG after creation — `--nsg-rule SSH` only opens port 22 by default.
5. Use `--storage-sku Standard_LRS` and `--os-disk-size-gb 30` for VMs in policy-restricted lab subscriptions.

---

## Outcome

| Task | Result |
|---|---|
| ACR datacenteracr30062 created (East US, Basic) | Done |
| Admin credentials enabled on ACR | Done |
| Docker image built locally (python:3.9-slim base) | Done |
| Image pushed to datacenter/python-app:latest | Done |
| Storage account datacenterstor30062 created (LRS) | Done |
| datacenter-config blob container created | Done |
| config.json uploaded to blob container | Done |
| datacenter-vm created (East US, B1s, Standard_LRS) | Done |
| Port 80 opened on VM NSG | Done |
| Docker and Azure CLI installed on VM | Done |
| Container pulled from ACR and running on port 80 | Done |
| config.json injected into running container | Done |
| curl http://20.124.0.57 → Welcome to KKE Azure Labs | Done |
