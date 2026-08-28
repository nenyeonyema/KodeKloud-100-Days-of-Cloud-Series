# AWS Task 38 — Containerized Application Deployment with ECR and ECS Fargate

## Overview

This project containerizes a Python application and deploys it on AWS using Amazon Elastic Container Registry (ECR) for image storage and Amazon Elastic Container Service (ECS) with Fargate as the serverless compute engine. The setup demonstrates a complete container delivery pipeline — from building a Docker image locally, pushing it to a private registry, and running it as a managed service on AWS without provisioning or managing any servers.

---

## Architecture

```
aws-client host
      │
      │ docker build
      ▼
Docker Image (xfusion-ecr:latest)
      │
      │ docker push
      ▼
ECR Private Repository (xfusion-ecr)
      │
      │ image reference in task definition
      ▼
ECS Task Definition (xfusion-taskdefinition)
      │
      │ runs on
      ▼
ECS Cluster (xfusion-cluster) — Fargate Launch Type
      │
      │ managed by
      ▼
ECS Service (xfusion-service)
[desired count: 1 | status: RUNNING]
```

---

## Prerequisites

- AWS CLI configured with appropriate credentials
- Docker installed and running on `aws-client` host
- Dockerfile located at `/root/pyapp/` on the `aws-client` host
- IAM permissions for ECR, ECS, and IAM in `us-east-1`

---

## Resources Created

| Resource | Name | Details |
|---|---|---|
| ECR Repository | `xfusion-ecr` | Private, AES256 encrypted, scan on push |
| ECS Cluster | `xfusion-cluster` | Fargate capacity provider |
| IAM Role | `xfusion-ecs-execution-role` | ECS task execution role |
| Task Definition | `xfusion-taskdefinition` | Fargate, 256 CPU, 512MB memory |
| ECS Service | `xfusion-service` | 1 desired task, awsvpc networking |

---

## Step-by-Step Setup

### 1. Set Up Variables

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
REGION="us-east-1"
ECR_REPO="xfusion-ecr"
ECR_URI="$ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com/$ECR_REPO"
echo "Account ID: $ACCOUNT_ID"
echo "ECR URI: $ECR_URI"
```

### 2. Create Private ECR Repository

```bash
aws ecr create-repository \
  --repository-name xfusion-ecr \
  --region $REGION \
  --image-scanning-configuration scanOnPush=true \
  --encryption-configuration encryptionType=AES256

echo "ECR repository created"

# Verify
aws ecr describe-repositories \
  --repository-names xfusion-ecr \
  --query "repositories[0].repositoryUri" \
  --output text
```

### 3. Build and Push Docker Image

```bash
# Authenticate Docker to ECR
aws ecr get-login-password --region $REGION | \
  docker login --username AWS --password-stdin \
  $ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com

# Build the image from the Dockerfile
cd /root/pyapp
docker build -t xfusion-ecr .
echo "Image built"

# Tag with ECR URI
docker tag xfusion-ecr:latest $ECR_URI:latest
echo "Image tagged"

# Push to ECR
docker push $ECR_URI:latest
echo "Image pushed"

# Verify image is in ECR
aws ecr describe-images \
  --repository-name xfusion-ecr \
  --query "imageDetails[*].{Tag:imageTags[0],Size:imageSizeInBytes,Pushed:imagePushedAt}" \
  --output table
```

### 4. Create ECS Cluster

```bash
aws ecs create-cluster \
  --cluster-name xfusion-cluster \
  --capacity-providers FARGATE \
  --default-capacity-provider-strategy \
    capacityProvider=FARGATE,weight=1 \
  --region $REGION

echo "ECS Cluster created"

# Verify
aws ecs describe-clusters \
  --clusters xfusion-cluster \
  --query "clusters[0].{Name:clusterName,Status:status}" \
  --output table
```

### 5. Create ECS Task Execution Role

```bash
cat > /tmp/ecs-trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "ecs-tasks.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }]
}
EOF

aws iam create-role \
  --role-name xfusion-ecs-execution-role \
  --assume-role-policy-document file:///tmp/ecs-trust-policy.json

# Attach AWS managed ECS task execution policy
aws iam attach-role-policy \
  --role-name xfusion-ecs-execution-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy

EXEC_ROLE_ARN="arn:aws:iam::${ACCOUNT_ID}:role/xfusion-ecs-execution-role"
echo "Execution Role: $EXEC_ROLE_ARN"
```

### 6. Create ECS Task Definition

```bash
# Write task definition using single-quoted heredoc to
# prevent bash from interpreting special characters
cat > /tmp/task-definition.json << 'ENDJSON'
{
  "family": "xfusion-taskdefinition",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "EXEC_ROLE_PLACEHOLDER",
  "containerDefinitions": [
    {
      "name": "xfusion-container",
      "image": "ECR_URI_PLACEHOLDER",
      "essential": true,
      "portMappings": [
        {
          "containerPort": 80,
          "hostPort": 80,
          "protocol": "tcp"
        }
      ]
    }
  ]
}
ENDJSON

# Substitute placeholders with actual values
sed -i "s|EXEC_ROLE_PLACEHOLDER|${EXEC_ROLE_ARN}|g" /tmp/task-definition.json
sed -i "s|ECR_URI_PLACEHOLDER|${ECR_URI}:latest|g" /tmp/task-definition.json

# Verify JSON content
cat /tmp/task-definition.json

# Register task definition
TASK_DEF_ARN=$(aws ecs register-task-definition \
  --cli-input-json file:///tmp/task-definition.json \
  --query "taskDefinition.taskDefinitionArn" \
  --output text)
echo "Task Definition ARN: $TASK_DEF_ARN"
```

> **Note:** Use a single-quoted heredoc `<< 'ENDJSON'` to prevent bash from expanding `!` and other special characters inside the JSON block. Use `sed` to substitute variables after the fact.

### 7. Create ECS Service

```bash
# Get default VPC and subnets
VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=isDefault,Values=true" \
  --query "Vpcs[0].VpcId" --output text)

SUBNET1=$(aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query "Subnets[0].SubnetId" --output text)
SUBNET2=$(aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query "Subnets[1].SubnetId" --output text)

DEFAULT_SG=$(aws ec2 describe-security-groups \
  --filters "Name=group-name,Values=default" "Name=vpc-id,Values=$VPC_ID" \
  --query "SecurityGroups[0].GroupId" --output text)

echo "VPC: $VPC_ID | SG: $DEFAULT_SG"
echo "Subnets: $SUBNET1, $SUBNET2"

# Open app port on default SG
aws ec2 authorize-security-group-ingress \
  --group-id $DEFAULT_SG \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0 2>/dev/null || echo "Port rule already exists"

# Create the ECS service
aws ecs create-service \
  --cluster xfusion-cluster \
  --service-name xfusion-service \
  --task-definition xfusion-taskdefinition \
  --desired-count 1 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={
    subnets=[$SUBNET1,$SUBNET2],
    securityGroups=[$DEFAULT_SG],
    assignPublicIp=ENABLED
  }" \
  --region $REGION

echo "ECS Service created"
```

---

## Verification

```bash
# Wait for service to stabilize
sleep 30

# Check service status
aws ecs describe-services \
  --cluster xfusion-cluster \
  --services xfusion-service \
  --query "services[0].{Status:status,Running:runningCount,Desired:desiredCount,Pending:pendingCount}" \
  --output table

# Get running task ARN
TASK_ARN=$(aws ecs list-tasks \
  --cluster xfusion-cluster \
  --service-name xfusion-service \
  --query "taskArns[0]" --output text)

# Check task status and IP
aws ecs describe-tasks \
  --cluster xfusion-cluster \
  --tasks $TASK_ARN \
  --query "tasks[0].{Status:lastStatus,Health:healthStatus,IP:containers[0].networkInterfaces[0].privateIpv4Address}" \
  --output table
```

**Expected output:**
```
+---------+-----------+-----------+---------+
| Desired |  Pending  |  Running  | Status  |
+---------+-----------+-----------+---------+
|  1      |  0        |  1        |  ACTIVE |
+---------+-----------+-----------+---------+

+---------+----------------+-----------+
| Health  |      IP        |  Status   |
+---------+----------------+-----------+
| UNKNOWN |  172.31.x.x    |  RUNNING  |
+---------+----------------+-----------+
```

> **Note:** `Health: UNKNOWN` is expected when no health check is defined in the task definition. It does not affect the running state of the container.

---

## How Fargate Works

```
You define:                    AWS manages:
─────────────────              ──────────────────────
Task Definition   ──────────▶  Server provisioning
(CPU, Memory,                  OS patching
 Image, Ports)                 Container runtime
                               Scaling infrastructure
ECS Service       ──────────▶  Task scheduling
(desired count,                Health monitoring
 networking)                   Auto-restart on failure
```

- **No EC2 instances to manage** — Fargate is fully serverless
- **Pay per task** — billed only for CPU and memory used while tasks run
- **awsvpc networking** — each task gets its own elastic network interface and private IP

---

## Security Highlights

- ECR repository is **private** — images are not publicly accessible
- **Image scanning on push** is enabled — vulnerabilities are detected automatically
- The ECS task execution role follows **least privilege** — only permissions needed to pull the image and write logs
- Container runs with **no SSH access** — managed entirely through ECS APIs

---

## Troubleshooting

| Issue | Cause | Fix |
|---|---|---|
| `Invalid JSON` on task definition | Bash expanding `!` or `$` in heredoc | Use single-quoted heredoc `<< 'EOF'` and `sed` for substitution |
| `TaskDefinition not found` on service create | Task definition registration failed silently | Check `TASK_DEF_ARN` is not empty before creating service |
| Service stuck in `PENDING` | Image pull failing from ECR | Verify execution role has `AmazonECSTaskExecutionRolePolicy` attached |
| `ECR auth token expired` | Docker login token lasts 12 hours | Re-run `aws ecr get-login-password` and `docker login` |
| Task immediately stops | Container crashing on startup | Check ECS task logs in CloudWatch under `/ecs/xfusion-taskdefinition` |

---

## Technologies Used

- **AWS ECR** — Private Docker image registry
- **AWS ECS** — Container orchestration service
- **AWS Fargate** — Serverless compute engine for containers
- **AWS IAM** — Task execution role for ECR and CloudWatch access
- **Docker** — Image build and push tooling
- **Python Application** — Containerized app from `/root/pyapp/Dockerfile`
- **AWS CLI** — Full infrastructure provisioning
