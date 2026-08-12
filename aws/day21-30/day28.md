# AWS Task 28 — Build and Push a Docker Image to Amazon ECR

## 📌 Task Overview

The Nautilus DevOps Team needed to establish a container registry workflow using Amazon ECR (Elastic Container Registry). The goal was to create a private ECR repository named `datacenter-ecr`, build a Docker image using a Python application Dockerfile located at `/root/pyapp`, authenticate Docker against AWS ECR, tag the image as `latest`, push it to the private registry, and verify the artifact availability in `us-east-1`.

| Requirement | Value |
| :--- | :--- |
| **ECR Repository Name** | `datacenter-ecr` (Private) |
| **Dockerfile Location** | `/root/pyapp` |
| **Image Tag** | `latest` |
| **AWS Region Context** | `us-east-1` |
| **Core Workflow** | Authenticate ──► Build ──► Tag ──► Push ──► Verify |

---

## 🎯 Architecture & Workflow

```text
               /root/pyapp (Dockerfile)
                          │
                          ▼
                    docker build
                          │
                          ▼
                 Local Docker Image
               (datacenter-ecr:latest)
                          │
                          │ docker tag
                          ▼
            Amazon ECR Repository URI
  <AWS_ACCOUNT_ID>[.dkr.ecr.us-east-1.amazonaws.com/datacenter-ecr:latest](https://.dkr.ecr.us-east-1.amazonaws.com/datacenter-ecr:latest)
                          │
                          │ docker login + docker push
                          ▼
              ┌───────────────────────┐
              │      Amazon ECR       │
              │    datacenter-ecr     │
              │                       │
              │     Image: latest     │
              └───────────────────────┘
🛠️ Implementation & Verification Commands
Bash
# ==========================================
# 1. CONFIGURE REGION & CREATE ECR REPOSITORY
# ==========================================
aws configure set region us-east-1

# Create Private ECR Repository
aws ecr create-repository \
  --repository-name datacenter-ecr \
  --region us-east-1

# Verify Repository Creation
aws ecr describe-repositories \
  --repository-names datacenter-ecr \
  --region us-east-1

# ==========================================
# 2. RESOLVE ECR URI & AUTHENTICATE DOCKER
# ==========================================
AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
ECR_REGISTRY="${AWS_ACCOUNT_ID}.dkr.ecr.us-east-1.amazonaws.com"
ECR_URI="${ECR_REGISTRY}/datacenter-ecr"

echo "Target ECR URI: ${ECR_URI}"

# Log in Docker to the ECR Registry
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin "$ECR_REGISTRY"

# ==========================================
# 3. BUILD, TAG, AND PUSH DOCKER IMAGE
# ==========================================
# Navigate to App Context and Inspect
cd /root/pyapp && ls -la

# Build Docker Image
docker build -t datacenter-ecr:latest /root/pyapp

# Tag Image for ECR Target
docker tag datacenter-ecr:latest "${ECR_URI}:latest"

# Push Image to Private ECR Repository
docker push "${ECR_URI}:latest"

# ==========================================
# 4. VERIFICATION & REGISTRY CHECKS
# ==========================================
# List Image Tags in Remote ECR Repository
aws ecr list-images \
  --repository-name datacenter-ecr \
  --region us-east-1

# Describe Detailed Image Metadata
aws ecr describe-images \
  --repository-name datacenter-ecr \
  --region us-east-1 \
  --query "imageDetails[0].{Digest:imageDigest, Tags:imageTags, SizeMB:imageSizeInBytes}" \
  --output table
Note: A returned image record matching tag latest from aws ecr describe-images confirms successful image build, authentication, layer upload, and availability inside the datacenter-ecr repository.
