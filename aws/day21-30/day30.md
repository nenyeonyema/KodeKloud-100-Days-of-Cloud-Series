# AWS Task 30 — NAT Instance for Private Subnet Internet Access

## 📌 Task Overview

The Nautilus DevOps Team required outbound internet connectivity for an isolated EC2 instance (`devops-priv-ec2`) residing in a private subnet (`devops-priv-subnet`) within `devops-priv-vpc`, without assigning a public IP address or exposing the workload to inbound internet traffic. To optimize operational costs compared to a managed NAT Gateway, a custom Amazon Linux 2 **NAT Instance** (`devops-nat-instance`) was deployed into a newly provisioned public subnet (`devops-pub-subnet`). Source/destination checks were disabled on the NAT instance, IP masquerading (`iptables`) was configured, and default routing (`0.0.0.0/0`) was updated in the private route table. Success was verified via automated S3 upload validation of `devops-test.txt` to bucket `devops-nat-8976`.

| Requirement | Value |
| :--- | :--- |
| **VPC / Subnets** | `devops-priv-vpc` ──► `devops-pub-subnet` (`10.0.2.0/24`) & `devops-priv-subnet` |
| **NAT Instance** | `devops-nat-instance` (Amazon Linux 2 / `t2.micro`) |
| **Security Group** | `devops-nat-sg` (All traffic allowed from VPC CIDR) |
| **Instance Attribute** | Source/Destination Check: `Disabled` (`false`) |
| **Networking Logic** | `sysctl net.ipv4.ip_forward=1` + `iptables -t nat MASQUERADE` |
| **Private Route Table** | `0.0.0.0/0` ──► `devops-nat-instance` |
| **S3 Verification Bucket** | `s3://devops-nat-8976/devops-test.txt` |
| **AWS Region Context** | `us-east-1` |

---

## 🎯 Architecture & Traffic Routing Flow

```text
                           Internet
                              │
                              ▼
                      Internet Gateway
                         (IGW-ID)
                              │
                              ▼
                     devops-pub-subnet
                       (10.0.2.0/24)
                              │
                              ▼
                     devops-nat-instance
                    (Source/Dest Check: false)
                              │
                              │ NAT / MASQUERADE (iptables)
                              ▼
                     devops-priv-subnet
                              │
                              ▼
                     devops-priv-ec2
                 (No Public IP Assigned)
                              │
                              │ Outbound S3 Request
                              ▼
                          S3 Bucket
                     (devops-nat-8976)
🛠️ Implementation & Verification Commands
Bash
# ==========================================
# 1. DISCOVER VPC & PRIVATE SUBNET
# ==========================================
aws configure set region us-east-1

# Resolve VPC ID & Private Subnet ID
VPC_ID=$(aws ec2 describe-vpcs --filters "Name=tag:Name,Values=devops-priv-vpc" --query "Vpcs[0].VpcId" --output text)
VPC_CIDR=$(aws ec2 describe-vpcs --vpc-ids "$VPC_ID" --query "Vpcs[0].CidrBlock" --output text)

PRIVATE_SUBNET_ID=$(aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID" "Name=tag:Name,Values=devops-priv-subnet" --query "Subnets[0].SubnetId" --output text)
PRIV_SUBNET_CIDR=$(aws ec2 describe-subnets --subnet-ids "$PRIVATE_SUBNET_ID" --query "Subnets[0].CidrBlock" --output text)

# ==========================================
# 2. CREATE PUBLIC SUBNET & INTERNET ROUTE
# ==========================================
# Create & Tag Public Subnet
PUBLIC_SUBNET_ID=$(aws ec2 create-subnet \
  --vpc-id "$VPC_ID" \
  --cidr-block 10.0.2.0/24 \
  --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=devops-pub-subnet}]' \
  --query "Subnet.SubnetId" \
  --output text)

aws ec2 modify-subnet-attribute --subnet-id "$PUBLIC_SUBNET_ID" --map-public-ip-on-launch

# Resolve/Attach Internet Gateway
IGW_ID=$(aws ec2 describe-internet-gateways --filters "Name=attachment.vpc-id,Values=$VPC_ID" --query "InternetGateways[0].InternetGatewayId" --output text)

if [ "$IGW_ID" == "None" ] \vert{}\vert{} [ -z "$IGW_ID" ]; then
  IGW_ID=$(aws ec2 create-internet-gateway --query "InternetGateway.InternetGatewayId" --output text)
  aws ec2 attach-internet-gateway --internet-gateway-id "$IGW_ID" --vpc-id "$VPC_ID"
fi

# Create & Associate Public Route Table
PUBLIC_RT_ID=$(aws ec2 create-route-table --vpc-id "$VPC_ID" --query "RouteTable.RouteTableId" --output text)
aws ec2 create-route --route-table-id "$PUBLIC_RT_ID" --destination-cidr-block 0.0.0.0/0 --gateway-id "$IGW_ID"
aws ec2 associate-route-table --route-table-id "$PUBLIC_RT_ID" --subnet-id "$PUBLIC_SUBNET_ID"

# ==========================================
# 3. CREATE NAT SECURITY GROUP & LAUNCH INSTANCE
# ==========================================
NAT_SG_ID=$(aws ec2 create-security-group \
  --group-name devops-nat-sg \
  --description "Security group for NAT instance" \
  --vpc-id "$VPC_ID" \
  --query "GroupId" \
  --output text)

# Allow Inbound Traffic from VPC CIDR & Outbound Everywhere
aws ec2 authorize-security-group-ingress --group-id "$NAT_SG_ID" --protocol -1 --cidr "$VPC_CIDR"

# Retrieve Amazon Linux 2 AMI ID
AMI_ID=$(aws ec2 describe-images \
  --owners amazon \
  --filters "Name=name,Values=amzn2-ami-hvm-*-x86_64-gp2" "Name=state,Values=available" \
  --query "sort_by(Images,&CreationDate)[-1].ImageId" \
  --output text)

# Launch NAT Instance in Public Subnet
NAT_INSTANCE_ID=$(aws ec2 run-instances \
  --image-id "$AMI_ID" \
  --instance-type t2.micro \
  --subnet-id "$PUBLIC_SUBNET_ID" \
  --security-group-ids "$NAT_SG_ID" \
  --associate-public-ip-address \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=devops-nat-instance}]' \
  --query "Instances[0].InstanceId" \
  --output text)

aws ec2 wait instance-running --instance-ids "$NAT_INSTANCE_ID"

# CRITICAL: Disable Source/Destination Checking on NAT Instance
aws ec2 modify-instance-attribute --instance-id "$NAT_INSTANCE_ID" --no-source-dest-check

# ==========================================
# 4. CONFIGURE NAT IP FORWARDING & IPTABLES
# ==========================================
# Execute IP Forwarding and NAT Masquerade rules on the NAT Instance via SSM Command
aws ssm send-command \
  --instance-ids "$NAT_INSTANCE_ID" \
  --document-name "AWS-RunShellScript" \
  --parameters commands="[
    'sudo sysctl -w net.ipv4.ip_forward=1',
    'sudo yum install -y iptables-services',
    'sudo systemctl enable iptables',
    'sudo systemctl start iptables',
    'sudo iptables -t nat -A POSTROUTING -o eth0 -s ${PRIV_SUBNET_CIDR} -j MASQUERADE',
    'sudo iptables -A FORWARD -s ${PRIV_SUBNET_CIDR} -j ACCEPT',
    'sudo iptables -A FORWARD -d ${PRIV_SUBNET_CIDR} -j ACCEPT',
    'sudo service iptables save'
  ]"

# ==========================================
# 5. UPDATE PRIVATE ROUTE TABLE TO POINT TO NAT INSTANCE
# ==========================================
PRIVATE_RT_ID=$(aws ec2 describe-route-tables --filters "Name=association.subnet-id,Values=$PRIVATE_SUBNET_ID" --query "RouteTables[0].RouteTableId" --output text)

# Add Default Route via NAT Instance
aws ec2 create-route \
  --route-table-id "$PRIVATE_RT_ID" \
  --destination-cidr-block 0.0.0.0/0 \
  --instance-id "$NAT_INSTANCE_ID"

# ==========================================
# 6. VERIFICATION & S3 UPLOAD CHECKS
# ==========================================
# Confirm SourceDestCheck state on NAT Instance (Expected: false)
aws ec2 describe-instances \
  --instance-ids "$NAT_INSTANCE_ID" \
  --query "Reservations[0].Instances[0].SourceDestCheck" \
  --output text

# Verify Private Subnet Route Table Default Entry
aws ec2 describe-route-tables \
  --route-table-ids "$PRIVATE_RT_ID" \
  --query "RouteTables[0].Routes[?DestinationCidrBlock=='0.0.0.0/0']" \
  --output table

# Verify S3 File Upload Initiated by Private Instance
aws s3api head-object \
  --bucket devops-nat-8976 \
  --key devops-test.txt
Note: A successful response from aws s3api head-object confirming devops-test.txt exists in s3://devops-nat-8976/ validates that the private instance successfully established outbound internet communication through devops-nat-instance.
