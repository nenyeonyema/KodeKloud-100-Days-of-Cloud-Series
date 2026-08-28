# AWS Task 43 — EKS Cluster Provisioning: Setting Up a Private Kubernetes Cluster on Amazon EKS

## Overview

This project documents the creation of a production-ready Amazon EKS cluster with a private endpoint, deployed across multiple availability zones using the default VPC. The team needed a secure, highly available Kubernetes cluster running the latest stable version, with EKS Auto Mode disabled and all access to the cluster API restricted to private network traffic only. An IAM role was created from scratch and attached to the cluster to meet internal security and scalability standards.

---

## Architecture

```
                        AWS Cloud (us-east-1)
┌──────────────────────────────────────────────────────────┐
│                                                          │
│   Default VPC                                            │
│   ┌──────────────────────────────────────────────────┐   │
│   │                                                  │   │
│   │  ┌─────────────┐ ┌─────────────┐ ┌───────────┐  │   │
│   │  │  us-east-1a │ │  us-east-1b │ │ us-east-1c│  │   │
│   │  │  (subnet)   │ │  (subnet)   │ │  (subnet) │  │   │
│   │  └──────┬──────┘ └──────┬──────┘ └─────┬─────┘  │   │
│   │         │               │              │         │   │
│   │         └───────────────┴──────────────┘         │   │
│   │                         │                        │   │
│   │                         ▼                        │   │
│   │              ┌─────────────────────┐             │   │
│   │              │   EKS Control Plane │             │   │
│   │              │    devops-eks       │             │   │
│   │              │                    │             │   │
│   │              │  Endpoint: PRIVATE  │             │   │
│   │              │  Public:   DISABLED │             │   │
│   │              │  AutoMode: DISABLED │             │   │
│   │              └─────────────────────┘             │   │
│   │                         │                        │   │
│   │                         ▼                        │   │
│   │              ┌─────────────────────┐             │   │
│   │              │   IAM Role          │             │   │
│   │              │   eksClusterRole    │             │   │
│   │              └─────────────────────┘             │   │
│   └──────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────┘
```

---

## Prerequisites

- AWS CLI configured with valid credentials
- IAM permissions for:
  - `eks:CreateCluster`
  - `iam:CreateRole`
  - `iam:AttachRolePolicy`
  - `iam:GetRole`
  - `ec2:DescribeVpcs`
  - `ec2:DescribeSubnets`
- Region: `us-east-1`

---

## What Was Built

An EKS cluster named `devops-eks` with:
- **IAM Role** — `eksClusterRole` created from scratch with required managed policies
- **Kubernetes version** — Latest stable (1.32)
- **VPC** — Default VPC with subnets across `us-east-1a`, `us-east-1b`, `us-east-1c`
- **Endpoint access** — Private only (`endpointPrivateAccess=true`, `endpointPublicAccess=false`)
- **EKS Auto Mode** — Disabled (not configured)

---

## IAM Role Configuration

```
eksClusterRole
    │
    ├── Trust Policy → eks.amazonaws.com (AssumeRole)
    │
    ├── AmazonEKSClusterPolicy        (required for EKS control plane)
    └── AmazonEKSVPCResourceController (required for VPC resource management)
```

---

## Step-by-Step Implementation

### 1. Verify AWS Credentials

```bash
aws sts get-caller-identity
```

### 2. Create IAM Trust Policy for EKS

```bash
cat > /tmp/eks-trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "eks.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

echo "✅ Trust policy created"
```

### 3. Create the eksClusterRole IAM Role

```bash
aws iam create-role \
  --role-name eksClusterRole \
  --assume-role-policy-document file:///tmp/eks-trust-policy.json \
  --description "IAM role for EKS cluster" \
  --region us-east-1

echo "✅ eksClusterRole created"
```

### 4. Attach Required Managed Policies

```bash
aws iam attach-role-policy \
  --role-name eksClusterRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSClusterPolicy

echo "✅ AmazonEKSClusterPolicy attached"

aws iam attach-role-policy \
  --role-name eksClusterRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSVPCResourceController

echo "✅ AmazonEKSVPCResourceController attached"
```

### 5. Get Role ARN

```bash
ROLE_ARN=$(aws iam get-role \
  --role-name eksClusterRole \
  --query 'Role.Arn' \
  --output text)

echo "Role ARN: $ROLE_ARN"
```

### 6. Get Default VPC and Subnets

```bash
# Get default VPC
VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=isDefault,Values=true" \
  --query 'Vpcs[0].VpcId' \
  --output text --region us-east-1)
echo "VPC ID: $VPC_ID"

# Get subnets for AZs a, b, c
SUBNET_IDS=$(aws ec2 describe-subnets \
  --filters \
    "Name=vpc-id,Values=$VPC_ID" \
    "Name=availabilityZone,Values=us-east-1a,us-east-1b,us-east-1c" \
  --query 'Subnets[*].SubnetId' \
  --output text --region us-east-1 | tr '\t' ',')
echo "Subnet IDs: $SUBNET_IDS"
```

### 7. Create the EKS Cluster

```bash
aws eks create-cluster \
  --name devops-eks \
  --role-arn "$ROLE_ARN" \
  --kubernetes-version "1.32" \
  --resources-vpc-config "subnetIds=$SUBNET_IDS,endpointPublicAccess=false,endpointPrivateAccess=true" \
  --region us-east-1 \
  --output json

echo "✅ EKS cluster creation initiated"
```

### 8. Poll Until Cluster is ACTIVE

```bash
for i in {1..30}; do
  STATUS=$(aws eks describe-cluster \
    --name devops-eks \
    --region us-east-1 \
    --query 'cluster.status' \
    --output text 2>/dev/null)
  echo "$(date '+%H:%M:%S') - Status: $STATUS"
  [ "$STATUS" == "ACTIVE" ] && echo "✅ Cluster is ACTIVE!" && break
  sleep 30
done
```

---

## Verification

**Confirm cluster configuration:**
```bash
aws eks describe-cluster \
  --name devops-eks \
  --region us-east-1 \
  --query 'cluster.{
    Name:name,
    Status:status,
    Version:version,
    RoleArn:roleArn,
    PublicAccess:resourcesVpcConfig.endpointPublicAccess,
    PrivateAccess:resourcesVpcConfig.endpointPrivateAccess,
    VpcId:resourcesVpcConfig.vpcId}' \
  --output table
```

**Confirm EKS Auto Mode is disabled:**
```bash
aws eks describe-cluster \
  --name devops-eks \
  --region us-east-1 \
  --query 'cluster.computeConfig' \
  --output json
```

**Expected output:**
```
---------------------------------------------------------------------
|                        DescribeCluster                            |
+-----------------+-------------------------------------------------+
| Name            | devops-eks                                      |
| Status          | ACTIVE                                          |
| Version         | 1.32                                            |
| PublicAccess    | False                                           |
| PrivateAccess   | True                                            |
| VpcId           | vpc-xxxxxxxxx                                   |
+-----------------+-------------------------------------------------+

computeConfig: null  ← EKS Auto Mode is DISABLED ✅
```

---

## Console Verification Steps

For those who prefer the AWS Console:

1. **EKS → Clusters** — confirm `devops-eks` cluster exists with status `Active`
2. **Cluster → Configuration → Networking tab** — confirm `Private endpoint` is `Enabled` and `Public endpoint` is `Disabled`
3. **Cluster → Configuration → Compute tab** — confirm EKS Auto Mode is not configured
4. **IAM → Roles** — confirm `eksClusterRole` exists with both managed policies attached
5. **Cluster → Configuration → Networking tab** — confirm subnets span `us-east-1a`, `us-east-1b`, `us-east-1c`

---

## Troubleshooting Checklist

| Check | Command | Expected Result |
|---|---|---|
| IAM role exists | `iam get-role` | Returns role ARN |
| Policies attached | `iam list-attached-role-policies` | Both EKS policies listed |
| Cluster exists | `eks list-clusters` | `devops-eks` listed |
| Cluster active | `eks describe-cluster` | `Status: ACTIVE` |
| Private endpoint | `eks describe-cluster` | `endpointPrivateAccess: true` |
| Public endpoint off | `eks describe-cluster` | `endpointPublicAccess: false` |
| Auto Mode off | `eks describe-cluster` | `computeConfig: null` |
| Correct subnets | `eks describe-cluster` | Subnets in 1a, 1b, 1c |

---

## Common Mistakes

| Mistake | Consequence |
|---|---|
| Not creating `eksClusterRole` before cluster creation | `NoSuchEntity` error — cluster creation fails immediately |
| Using tab-separated subnet IDs from `--output text` without converting | AWS CLI receives broken parameters — `Unknown options` error |
| Not quoting `--resources-vpc-config` value | Shell splits the argument on spaces — CLI rejects it |
| Using an outdated Kubernetes version | Cluster may not support latest EKS features |
| Skipping the IAM policy attachments | EKS control plane cannot manage VPC resources — cluster stays in `CREATING` indefinitely |

---

## Key Lesson

When passing multiple values to `--resources-vpc-config`, always **quote the entire value as a single string** and ensure subnet IDs are **comma-separated not tab-separated**. The AWS CLI `--output text` flag returns tab-delimited values by default — always pipe through `tr '\t' ','` before using them in another command.

Always follow this order when setting up EKS:

```
Create IAM Role → Attach Policies → Get VPC & Subnets
→ Create Cluster → Poll for ACTIVE → Verify endpoint config
```

Never attempt cluster creation without first confirming the IAM role exists and has both required policies attached — the cluster will fail immediately with no useful error message.

---

## Technologies Used

- **Amazon EKS** — Managed Kubernetes control plane service
- **AWS IAM** — Role and policy management for EKS access
- **Amazon VPC** — Default VPC with multi-AZ subnet distribution
- **AWS CLI** — Cluster creation, status polling and verification
- **Kubernetes 1.32** — Latest stable version at time of deployment
