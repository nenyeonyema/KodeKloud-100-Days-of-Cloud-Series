# AWS Task 26 — EC2 Web Server Provisioning with Nginx and User Data

## 📌 Task Overview

The Nautilus DevOps Team required a web server automated via EC2 User Data bootstrap scripting. The goal was to provision an Ubuntu 24.04 LTS EC2 instance (`nautilus-ec2`), create a custom security group (`nautilus-web-sg`) opening port 80 to the internet, automatically install and start the Nginx web service during launch, and verify web server availability via the instance's public IP address in `us-east-1`.

| Requirement | Value |
| :--- | :--- |
| **EC2 Instance Name** | `nautilus-ec2` |
| **AMI / Instance Type** | Ubuntu 24.04 LTS / `t2.micro` |
| **Security Group** | `nautilus-web-sg` (HTTP:80 from `0.0.0.0/0`) |
| **Bootstrap Script** | EC2 User Data (`user-data.sh`) |
| **Service Installed** | `nginx` (Enabled & Started) |
| **AWS Region Context** | `us-east-1` |

---

## 🎯 Architecture & Execution Flow

```text
                    Internet
                        │
                        │ HTTP :80
                        ▼
              ┌───────────────────┐
              │  Security Group   │
              │  nautilus-web-sg  │
              └─────────┬─────────┘
                        │
                        ▼
              ┌───────────────────┐
              │   nautilus-ec2    │
              │   Ubuntu Server   │
              └─────────┬─────────┘
                        │ User Data Bootstrapping
                        ▼
              ┌───────────────────┐
              │  apt update/install│
              │  systemctl nginx  │
              └───────────────────┘
🛠️ Implementation & Verification Commands
Bash
# ==========================================
# 1. CONFIGURE REGION & DISCOVER NETWORKING/AMI
# ==========================================
aws configure set region us-east-1

# Discover latest Ubuntu 24.04 LTS AMI
AMI_ID=$(aws ec2 describe-images \
  --owners 099720109477 \
  --filters \
    "Name=name,Values=ubuntu/images/hvm-ssd-gp3/ubuntu-noble-24.04-amd64-server-*" \
    "Name=state,Values=available" \
    "Name=architecture,Values=x86_64" \
  --query "sort_by(Images,&CreationDate)[-1].ImageId" \
  --output text)

# Get Default VPC and Subnet IDs
VPC_ID=$(aws ec2 describe-vpcs --filters "Name=is-default,Values=true" --query "Vpcs[0].VpcId" --output text)
SUBNET_ID=$(aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID" --query "Subnets[0].SubnetId" --output text)

# ==========================================
# 2. CREATE SECURITY GROUP & ALLOW HTTP
# ==========================================
SG_ID=$(aws ec2 create-security-group \
  --group-name nautilus-web-sg \
  --description "Security group for Nautilus web server" \
  --vpc-id "$VPC_ID" \
  --query "GroupId" \
  --output text)

aws ec2 authorize-security-group-ingress \
  --group-id "$SG_ID" \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0

# ==========================================
# 3. CREATE BOOTSTRAP USER DATA SCRIPT
# ==========================================
cat > user-data.sh <<'EOF'
#!/bin/bash
apt-get update -y
apt-get install -y nginx
systemctl enable nginx
systemctl start nginx
EOF

# ==========================================
# 4. LAUNCH & TAG INSTANCE WITH USER DATA
# ==========================================
INSTANCE_ID=$(aws ec2 run-instances \
  --image-id "$AMI_ID" \
  --instance-type t2.micro \
  --subnet-id "$SUBNET_ID" \
  --security-group-ids "$SG_ID" \
  --user-data file://user-data.sh \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=nautilus-ec2}]' \
  --query "Instances[0].InstanceId" \
  --output text)

# Wait for instance to reach running state
aws ec2 wait instance-running --instance-ids "$INSTANCE_ID"

# ==========================================
# 5. VERIFICATION & HTTP ENDPOINT CHECK
# ==========================================
# Retrieve Public IP
PUBLIC_IP=$(aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query "Reservations[0].Instances[0].PublicIpAddress" \
  --output text)

echo "EC2 Instance ID: $INSTANCE_ID \vert{} Public IP:$PUBLIC_IP"

# Test HTTP access to verify Nginx execution via User Data
curl -I "http://$PUBLIC_IP"
Note: A returned HTTP/1.1 200 OK header response from curl -I http://$PUBLIC_IP verifies that the User Data script executed successfully upon launch, Nginx is active, and port 80 is publicly accessible.
