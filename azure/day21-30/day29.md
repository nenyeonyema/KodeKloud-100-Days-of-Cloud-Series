# Task 29 — Create Azure Container Registry and Push Docker Image

## Overview

The Nautilus DevOps Team needed to create an Azure Container Registry (ACR) to store Docker images for their containerized applications. A Dockerfile already existed on the azure-client host under `/root/pyapp`. The task involved creating the ACR repository, building a Docker image from the Dockerfile, and pushing it to the registry with the tag `latest`.

---

## Requirements

| Parameter | Value |
|---|---|
| ACR Name | `devopsacr20125` |
| Pricing Plan | Basic |
| Region | East US |
| Dockerfile Location | `/root/pyapp/` on azure-client |
| Image Tag | `devopsacr20125:latest` |
| Login Server | `devopsacr20125.azurecr.io` |

---

## Prerequisites

- Azure CLI installed and configured on the azure-client host
- Docker installed and running on the azure-client host
- Dockerfile exists at `/root/pyapp/Dockerfile`
- Access to Azure credentials via `showcreds` command

---

## Steps Performed

### 1. Login to Azure
```bash
az login -u "<username>" -p "<password>"
az group list --output table
```

### 2. Inspect the Dockerfile
```bash
ls /root/pyapp/
cat /root/pyapp/Dockerfile
```

### 3. Create the ACR Repository
```bash
az acr create \
    --resource-group kml_rg_main-408c497fc64f4f6a \
    --name devopsacr20125 \
    --sku Basic \
    --location eastus

echo "ACR created!"
```

### 4. Enable Admin Access on ACR
```bash
az acr update \
    --name devopsacr20125 \
    --admin-enabled true
```

> Admin access must be enabled to allow Docker to authenticate with the registry using username/password credentials.

### 5. Get ACR Login Credentials
```bash
az acr credential show \
    --name devopsacr20125 \
    --query "{Username:username, Password:passwords[0].value}" \
    -o table

ACR_PASSWORD=$(az acr credential show \
    --name devopsacr20125 \
    --query "passwords[0].value" -o tsv)
```

### 6. Login to ACR with Docker
```bash
docker login devopsacr20125.azurecr.io \
    --username devopsacr20125 \
    --password $ACR_PASSWORD
```

### 7. Build the Docker Image
```bash
docker build -t devopsacr20125.azurecr.io/devopsacr20125:latest /root/pyapp/
```

### 8. Push the Image to ACR
```bash
docker push devopsacr20125.azurecr.io/devopsacr20125:latest
```

### 9. Verify Image in Registry
```bash
az acr repository list \
    --name devopsacr20125 \
    --output table

az acr repository show-tags \
    --name devopsacr20125 \
    --repository devopsacr20125 \
    --output table
```

Expected output:
```
Result
--------
latest
```

---

## Issues Encountered & Fixes

| Issue | Cause | Fix |
|---|---|---|
| `AuthorizationFailed` on `az acr build` | Lab user account lacked `Microsoft.ContainerRegistry/registries/scheduleRun/action` permission | Switched to local Docker build + push instead of `az acr build` |
| `az acr build` is not always available | Requires elevated ACR permissions not granted in lab environments | Use `docker build` + `docker login` + `docker push` workflow instead |

---

## Two Approaches Compared

### Approach 1 — `az acr build` (Cloud Build) — Failed in Lab
```bash
az acr build \
    --registry devopsacr20125 \
    --image devopsacr20125:latest \
    /root/pyapp/
```
- Sends build context to ACR and builds in the cloud
- No local Docker daemon needed
- **Requires `scheduleRun` permission** — not available in this lab

### Approach 2 — Local Docker Build + Push — Used Successfully
```bash
docker build -t devopsacr20125.azurecr.io/devopsacr20125:latest /root/pyapp/
docker push devopsacr20125.azurecr.io/devopsacr20125:latest
```
- Builds image locally on azure-client
- Pushes to ACR using Docker credentials
- Works with admin-enabled ACR authentication

---

## Verification Output

```
Result
--------
devopsacr20125

Tags:
--------
latest
```

| Item | Value |
|---|---|
| ACR Name | `devopsacr20125` ✅ |
| SKU | Basic ✅ |
| Region | East US ✅ |
| Image | `devopsacr20125.azurecr.io/devopsacr20125:latest` ✅ |
| Tag | `latest` ✅ |

---

## Key Concepts

**What is Azure Container Registry (ACR)?**
ACR is Azure's private Docker registry service — similar to Docker Hub but private and integrated with Azure services like AKS, Azure Container Instances, and Azure DevOps. It stores and manages Docker container images and Helm charts.

**ACR SKUs — Basic vs Standard vs Premium**
| SKU | Use Case | Storage | Webhooks |
|---|---|---|---|
| Basic | Dev/test, small teams | 10 GB | Yes |
| Standard | Production workloads | 100 GB | Yes |
| Premium | Geo-replication, private endpoints | 500 GB | Yes |

Basic is the most cost-effective for lab and development scenarios.

**Why Enable Admin Access?**
By default, ACR uses Azure AD authentication (service principals, managed identities). Admin access provides a simple username/password fallback that Docker's `docker login` command can use directly. In production, service principals or managed identities are preferred over admin credentials.

**ACR Image Naming Convention**
Images pushed to ACR follow this format:
```
<registry-name>.azurecr.io/<repository-name>:<tag>
```
For this task:
```
devopsacr20125.azurecr.io/devopsacr20125:latest
```
The `<registry-name>` prefix in the image tag tells Docker where to push the image.

**`az acr build` vs `docker build + push`**
`az acr build` is more convenient (no local Docker needed) but requires the `AcrPush` or `Contributor` role on the registry. When those permissions are unavailable, the standard Docker workflow (`build` → `login` → `push`) is the reliable fallback.
