# AWS Task 29 — VPC Peering Between Public and Private VPCs

## 📌 Task Overview

The Nautilus DevOps Team needed to establish private, non-transitive inter-VPC network connectivity between a public-facing instance (`devops-public-ec2`) in the default VPC and a isolated instance (`devops-private-ec2`) residing inside `devops-private-vpc` (`10.1.0.0/16`). A VPC peering connection (`devops-vpc-peering`) was created and accepted, cross-VPC routes were injected into both respective route tables, ICMP security group rules were configured, and connectivity was validated via direct ICMP ping.

| Requirement | Value |
| :--- | :--- |
| **Public VPC / EC2** | Default VPC ──► `devops-public-ec2` |
| **Private VPC / CIDR** | `devops-private-vpc` (`10.1.0.0/16`) |
| **Private Subnet / CIDR** | `devops-private-subnet` (`10.1.1.0/24`) |
| **Private EC2** | `devops-private-ec2` |
| **Peering Connection** | `devops-vpc-peering` (State: `active`) |
| **Ingress Access** | ICMP Echo Requests allowed from Default VPC CIDR |
| **AWS Region Context** | `us-east-1` |

---

## 🎯 Architecture & Network Flow

```text
                        Internet
                           │
                           ▼
                 ┌───────────────────┐
                 │    Default VPC    │
                 │  devops-public    │
                 │       -ec2        │
                 └─────────┬─────────┘
                           │
                           │ Private Traffic (No Internet Hop)
                           ▼
                  VPC Peering Connection
                   (devops-vpc-peering)
                           │
                           ▼
                 ┌───────────────────┐
                 │ devops-private-vpc│
                 │   (10.1.0.0/16)   │
                 │                   │
                 │  devops-private   │
                 │       -subnet     │ (10.1.1.0/24)
                 │         │         │
                 │         ▼         │
                 │  devops-private   │
                 │       -ec2        │
                 └───────────────────┘
🛠️ Implementation & Verification Commands
Bash
# ==========================================
# 1. DISCOVER VPCS & EC2 INSTANCES
# ==========================================
aws configure set region us-east-1

# Identify Default VPC & Private VPC
DEFAULT_VPC_ID=$(aws ec2 describe-vpcs --filters "Name=is-default,Values=true" --query "Vpcs[0].VpcId" --output text)
PRIVATE_VPC_ID=$(aws ec2 describe-vpcs --filters "Name=tag:Name,Values=devops-private-vpc" --query "Vpcs[0].VpcId" --output text)
DEFAULT_VPC_CIDR=$(aws ec2 describe-vpcs --vpc-ids "$DEFAULT_VPC_ID" --query "Vpcs[0].CidrBlock" --output text)

# Locate Instances & Get Private IP
PUBLIC_INSTANCE_ID=$(aws ec2 describe-instances --filters "Name=tag:Name,Values=devops-public-ec2" --query "Reservations[0].Instances[0].InstanceId" --output text)
PRIVATE_INSTANCE_ID=$(aws ec2 describe-instances --filters "Name=tag:Name,Values=devops-private-ec2" --query "Reservations[0].Instances[0].InstanceId" --output text)

PRIVATE_IP=$(aws ec2 describe-instances --instance-ids "$PRIVATE_INSTANCE_ID" --query "Reservations[0].Instances[0].PrivateIpAddress" --output text)

# ==========================================
# 2. CREATE & ACCEPT VPC PEERING CONNECTION
# ==========================================
PEERING_ID=$(aws ec2 create-vpc-peering-connection \
  --vpc-id "$DEFAULT_VPC_ID" \
  --peer-vpc-id "$PRIVATE_VPC_ID" \
  --tag-specifications 'ResourceType=vpc-peering-connection,Tags=[{Key=Name,Value=devops-vpc-peering}]' \
  --query "VpcPeeringConnection.VpcPeeringConnectionId" \
  --output text)

# Accept Peering Request (Same account / region)
aws ec2 accept-vpc-peering-connection --vpc-peering-connection-id "$PEERING_ID"

# ==========================================
# 3. CONFIGURE BI-DIRECTIONAL ROUTE TABLES
# ==========================================
# Discover Route Tables
PUBLIC_SUBNET_ID=$(aws ec2 describe-instances --instance-ids "$PUBLIC_INSTANCE_ID" --query "Reservations[0].Instances[0].SubnetId" --output text)
PUBLIC_RT_ID=$(aws ec2 describe-route-tables --filters "Name=association.subnet-id,Values=$PUBLIC_SUBNET_ID" --query "RouteTables[0].RouteTableId" --output text)

PRIVATE_SUBNET_ID=$(aws ec2 describe-subnets --filters "Name=vpc-id,Values=$PRIVATE_VPC_ID" "Name=tag:Name,Values=devops-private-subnet" --query "Subnets[0].SubnetId" --output text)
PRIVATE_RT_ID=$(aws ec2 describe-route-tables --filters "Name=association.subnet-id,Values=$PRIVATE_SUBNET_ID" --query "RouteTables[0].RouteTableId" --output text)

# Route Public VPC (10.1.0.0/16 -> Peering)
aws ec2 create-route \
  --route-table-id "$PUBLIC_RT_ID" \
  --destination-cidr-block 10.1.0.0/16 \
  --vpc-peering-connection-id "$PEERING_ID"

# Route Private VPC (Default VPC CIDR -> Peering)
aws ec2 create-route \
  --route-table-id "$PRIVATE_RT_ID" \
  --destination-cidr-block "$DEFAULT_VPC_CIDR" \
  --vpc-peering-connection-id "$PEERING_ID"

# ==========================================
# 4. CONFIGURE SECURITY GROUP FOR ICMP
# ==========================================
PRIVATE_SG_ID=$(aws ec2 describe-instances --instance-ids "$PRIVATE_INSTANCE_ID" --query "Reservations[0].Instances[0].SecurityGroups[0].GroupId" --output text)

# Allow ICMP Traffic from Default VPC CIDR
aws ec2 authorize-security-group-ingress \
  --group-id "$PRIVATE_SG_ID" \
  --protocol icmp \
  --port -1 \
  --cidr "$DEFAULT_VPC_CIDR"

# ==========================================
# 5. SSH ACCESS & CONNECTIVITY TESTING
# ==========================================
# Fetch Public Instance Public IP
PUBLIC_IP=$(aws ec2 describe-instances --instance-ids "$PUBLIC_INSTANCE_ID" --query "Reservations[0].Instances[0].PublicIpAddress" --output text)

# Connect to Public Instance and Ping Private Instance
ssh -o StrictHostKeyChecking=no ec2-user@"$PUBLIC_IP" "ping -c 4 $PRIVATE_IP"
Note: Successful 0% packet loss response from ping -c 4 $PRIVATE_IP confirms that the Peering Connection status is active, routing entries are properly propagated, and Security Group ingress rules permit cross-VPC private communication.
