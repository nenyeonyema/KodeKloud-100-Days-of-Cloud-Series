# AWS Task 24 — Setting Up an Application Load Balancer for an EC2 Instance

## 📌 Task Overview

The Nautilus DevOps Team needed to place an Application Load Balancer (ALB) in front of an existing EC2 instance running Nginx (`devops-ec2`). The objective was to create a dedicated security group, provision an ALB (`devops-alb`) and target group (`devops-tg`), configure traffic forwarding on port 80, restrict EC2 access to only allow traffic originating from the ALB, and verify application delivery via the ALB DNS name in `us-east-1`.

| Requirement | Value |
| :--- | :--- |
| **ALB Name** | `devops-alb` |
| **Target Group** | `devops-tg` (HTTP:80) |
| **ALB Security Group** | `devops-sg` (HTTP:80 from `0.0.0.0/0`) |
| **Target EC2 Instance** | `devops-ec2` |
| **AWS Region Context** | `us-east-1` |
| **Traffic Path** | Internet ──► `devops-alb` ──► `devops-tg` ──► `devops-ec2` (Nginx) |

---

## 🎯 Architecture & Traffic Flow

```text
                       Internet
                           │
                           │ TCP :80
                           ▼
                  ┌───────────────────┐
                  │    devops-sg      │ (Inbound: 0.0.0.0/0)
                  └─────────┬─────────┘
                            │
                            ▼
                  ┌───────────────────┐
                  │    devops-alb     │ (Application Load Balancer)
                  └─────────┬─────────┘
                            │ Forward :80
                            ▼
                  ┌───────────────────┐
                  │     devops-tg     │ (Target Group)
                  └─────────┬─────────┘
                            │ HTTP :80
                            ▼
                  ┌───────────────────┐
                  │  EC2 Sec Group    │ (Inbound: devops-sg)
                  └─────────┬─────────┘
                            │
                            ▼
                  ┌───────────────────┐
                  │    devops-ec2     │ (Nginx Web Server)
                  └───────────────────┘
🛠️ Implementation & Verification Commands
Bash
# ==========================================
# 1. CONFIGURE REGION & DISCOVER INSTANCE/VPC
# ==========================================
aws configure set region us-east-1

# Get EC2 Instance ID, VPC ID, and existing Security Group ID
INSTANCE_ID=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=devops-ec2" "Name=instance-state-name,Values=running" \
  --query "Reservations[0].Instances[0].InstanceId" \
  --output text)

VPC_ID=$(aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query "Reservations[0].Instances[0].VpcId" \
  --output text)

EC2_SG_ID=$(aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query "Reservations[0].Instances[0].SecurityGroups[0].GroupId" \
  --output text)

echo "Instance ID: $INSTANCE_ID | VPC ID: $VPC_ID \vert{} EC2 SG ID:$EC2_SG_ID"

# ==========================================
# 2. CONFIGURE SECURITY GROUPS
# ==========================================
# Create security group for ALB
ALB_SG_ID=$(aws ec2 create-security-group \
  --group-name devops-sg \
  --description "Security group for devops ALB" \
  --vpc-id "$VPC_ID" \
  --query "GroupId" \
  --output text)

# Allow HTTP on port 80 to the ALB from anywhere
aws ec2 authorize-security-group-ingress \
  --group-id "$ALB_SG_ID" \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0

# Allow traffic into EC2 instance ONLY from the ALB security group
aws ec2 authorize-security-group-ingress \
  --group-id "$EC2_SG_ID" \
  --protocol tcp \
  --port 80 \
  --source-group "$ALB_SG_ID"

# ==========================================
# 3. CREATE TARGET GROUP & REGISTER INSTANCE
# ==========================================
TARGET_GROUP_ARN=$(aws elbv2 create-target-group \
  --name devops-tg \
  --protocol HTTP \
  --port 80 \
  --vpc-id "$VPC_ID" \
  --target-type instance \
  --health-check-protocol HTTP \
  --health-check-port traffic-port \
  --health-check-path / \
  --query "TargetGroups[0].TargetGroupArn" \
  --output text)

aws elbv2 register-targets \
  --target-group-arn "$TARGET_GROUP_ARN" \
  --targets Id="$INSTANCE_ID",Port=80

# ==========================================
# 4. PROVISION ALB & CREATE LISTENER
# ==========================================
# Discover two subnets across distinct Availability Zones
SUBNET_1=$(aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID" --query "Subnets[0].SubnetId" --output text)
SUBNET_2=$(aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID" --query "Subnets[1].SubnetId" --output text)

ALB_ARN=$(aws elbv2 create-load-balancer \
  --name devops-alb \
  --subnets "$SUBNET_1" "$SUBNET_2" \
  --security-groups "$ALB_SG_ID" \
  --scheme internet-facing \
  --type application \
  --ip-address-type ipv4 \
  --query "LoadBalancers[0].LoadBalancerArn" \
  --output text)

# Create listener to forward port 80 traffic to the target group
aws elbv2 create-listener \
  --load-balancer-arn "$ALB_ARN" \
  --protocol HTTP \
  --port 80 \
  --default-actions Type=forward,TargetGroupArn="$TARGET_GROUP_ARN"

# ==========================================
# 5. VERIFICATION & HEALTH CHECKS
# ==========================================
# Verify Load Balancer state (Expected: active)
aws elbv2 describe-load-balancers \
  --load-balancer-arns "$ALB_ARN" \
  --query "LoadBalancers[0].State.Code" \
  --output text

# Verify Target Health state (Expected: healthy)
aws elbv2 describe-target-health \
  --target-group-arn "$TARGET_GROUP_ARN" \
  --query "TargetHealthDescriptions[].{Instance:Target.Id,Port:Target.Port,State:TargetHealth.State}" \
  --output table

# Retrieve ALB DNS Name and test application access
ALB_DNS=$(aws elbv2 describe-load-balancers \
  --load-balancer-arns "$ALB_ARN" \
  --query "LoadBalancers[0].DNSName" \
  --output text)

echo "Testing ALB Endpoint: http://$ALB_DNS"
curl -I "http://$ALB_DNS"
Note: A successful HTTP 200 OK response from curl http://$ALB_DNS and a target state of healthy confirm that the ALB is properly routing traffic to Nginx on the backend EC2 instance.
