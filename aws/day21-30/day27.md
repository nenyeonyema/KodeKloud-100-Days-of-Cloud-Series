# AWS Task 27 — Public VPC, Subnet, and EC2 Instance Provisioning

## 📌 Task Overview

The Nautilus DevOps Team required a public Amazon VPC infrastructure to host public-facing workloads. The objective was to construct a custom VPC (`devops-pub-vpc`), provision a public subnet (`devops-pub-subnet`) with auto-assign public IPv4 enabled, attach an Internet Gateway (`devops-pub-igw`), configure custom route tables (`devops-pub-rt`) pointing `0.0.0.0/0` to the IGW, create a dedicated security group (`devops-pub-sg`) allowing SSH access on port 22, and launch a publicly accessible Ubuntu EC2 instance (`devops-pub-ec2`) in `us-east-1`.

| Requirement | Value |
| :--- | :--- |
| **VPC Name / CIDR** | `devops-pub-vpc` (`10.0.0.0/16`) |
| **Subnet Name / CIDR** | `devops-pub-subnet` (`10.0.1.0/24`) in `us-east-1a` |
| **Internet Gateway** | `devops-pub-igw` (Attached) |
| **Route Table** | `devops-pub-rt` (`0.0.0.0/0` ──► `devops-pub-igw`) |
| **Auto-Assign Public IP** | Enabled (`true`) |
| **Security Group** | `devops-pub-sg` (SSH:22 from `0.0.0.0/0`) |
| **EC2 Instance / AMI** | `devops-pub-ec2` (Ubuntu 24.04 LTS / `t2.micro`) |
| **AWS Region Context** | `us-east-1` |

---

## 🎯 Architecture & Routing Flow

```text
                    Internet
                       │
                       │ SSH :22
                       ▼
               Internet Gateway
               (devops-pub-igw)
                       │
                       ▼
            ┌─────────────────────┐
            │   devops-pub-vpc    │
            │    10.0.0.0/16      │
            │                     │
            │  devops-pub-subnet  │ (10.0.1.0/24)
            │  (Auto-Assign IP)   │
            │          │          │
            │          ▼          │
            │   devops-pub-sg     │ (Inbound TCP :22)
            │          │          │
            │          ▼          │
            │   devops-pub-ec2    │
            │     t2.micro        │
            └─────────────────────┘
🛠️ Implementation & Verification Commands
Bash
# ==========================================
# 1. CONFIGURE REGION & CREATE VPC / SUBNET
# ==========================================
aws configure set region us-east-1

# Create and Tag Custom VPC
VPC_ID=$(aws ec2 create-vpc \
  --cidr-block 10.0.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=devops-pub-vpc}]' \
  --query "Vpc.VpcId" \
  --output text)

# Create and Tag Public Subnet
SUBNET_ID=$(aws ec2 create-subnet \
  --vpc-id "$VPC_ID" \
  --cidr-block 10.0.1.0/24 \
  --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=devops-pub-subnet}]' \
  --query "Subnet.SubnetId" \
  --output text)

# Enable Auto-Assign Public IP on Subnet
aws ec2 modify-subnet-attribute \
  --subnet-id "$SUBNET_ID" \
  --map-public-ip-on-launch

# ==========================================
# 2. PROVISION INTERNET GATEWAY & ROUTE TABLE
# ==========================================
# Create, Tag, and Attach Internet Gateway
IGW_ID=$(aws ec2 create-internet-gateway \
  --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=devops-pub-igw}]' \
  --query "InternetGateway.InternetGatewayId" \
  --output text)

aws ec2 attach-internet-gateway --internet-gateway-id "$IGW_ID" --vpc-id "$VPC_ID"

# Create, Tag, and Configure Public Route Table
RT_ID=$(aws ec2 create-route-table \
  --vpc-id "$VPC_ID" \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=devops-pub-rt}]' \
  --query "RouteTable.RouteTableId" \
  --output text)

# Add Default Route (0.0.0.0/0) to IGW
aws ec2 create-route \
  --route-table-id "$RT_ID" \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id "$IGW_ID"

# Associate Route Table with Subnet
aws ec2 associate-route-table --route-table-id "$RT_ID" --subnet-id "$SUBNET_ID"

# ==========================================
# 3. CREATE SECURITY GROUP & AUTHORIZE SSH
# ==========================================
SG_ID=$(aws ec2 create-security-group \
  --group-name devops-pub-sg \
  --description "Security group for public EC2 instance" \
  --vpc-id "$VPC_ID" \
  --query "GroupId" \
  --output text)

aws ec2 authorize-security-group-ingress \
  --group-id "$SG_ID" \
  --protocol tcp \
  --port 22 \
  --cidr 0.0.0.0/0

# ==========================================
# 4. DISCOVER AMI & LAUNCH EC2 INSTANCE
# ==========================================
AMI_ID=$(aws ec2 describe-images \
  --owners 099720109477 \
  --filters \
    "Name=name,Values=ubuntu/images/hvm-ssd-gp3/ubuntu-noble-24.04-amd64-server-*" \
    "Name=state,Values=available" \
    "Name=architecture,Values=x86_64" \
  --query "sort_by(Images,&CreationDate)[-1].ImageId" \
  --output text)

INSTANCE_ID=$(aws ec2 run-instances \
  --image-id "$AMI_ID" \
  --instance-type t2.micro \
  --subnet-id "$SUBNET_ID" \
  --security-group-ids "$SG_ID" \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=devops-pub-ec2}]' \
  --query "Instances[0].InstanceId" \
  --output text)

# Wait until instance is running
aws ec2 wait instance-running --instance-ids "$INSTANCE_ID"

# ==========================================
# 5. VERIFICATION & INTEGRITY CHECKS
# ==========================================
# Verify Auto-Public IP setting on Subnet (Expected: true)
aws ec2 describe-subnets \
  --subnet-ids "$SUBNET_ID" \
  --query "Subnets[0].{SubnetId:SubnetId, CIDR:CidrBlock, MapPublicIpOnLaunch:MapPublicIpOnLaunch}" \
  --output table

# Verify Default Route through IGW (Expected: 0.0.0.0/0 -> igw-xxxx)
aws ec2 describe-route-tables \
  --route-table-ids "$RT_ID" \
  --query "RouteTables[0].Routes[?DestinationCidrBlock=='0.0.0.0/0']" \
  --output table

# Verify EC2 Instance Public IP Allocation
aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query "Reservations[0].Instances[0].{InstanceId:InstanceId, State:State.Name, PublicIP:PublicIpAddress, PrivateIP:PrivateIpAddress}" \
  --output table
Note: The presence of a public IPv4 address on devops-pub-ec2, combined with a true status for MapPublicIpOnLaunch and a verified 0.0.0.0/0 route to devops-pub-igw, confirms that the VPC architecture is publicly accessible.
