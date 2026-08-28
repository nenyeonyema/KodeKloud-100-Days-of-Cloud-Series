# AWS Task 40 — VPC Troubleshooting: Restoring Internet Access to EC2 Instance

## Overview

This project documents the troubleshooting and resolution of an internet connectivity issue affecting an EC2 instance running Nginx inside a public VPC. Despite the security group being correctly configured to allow traffic on port 80, the application was inaccessible from the internet. The root cause was identified as a misconfigured VPC — specifically a detached Internet Gateway and a missing route table entry — leaving the VPC completely isolated from the internet.

---

## Architecture

### Before Fix (Broken)
```
Internet
    │
    ✖ No path exists
    │
devops-vpc (isolated)
    │
    └── devops-ec2 (Nginx running but unreachable)
```

### After Fix (Working)
```
Internet
    │
    ▼ port 80
Internet Gateway (devops-igw)
[attached to devops-vpc]
    │
    ▼ route: 0.0.0.0/0 → igw
Route Table
    │
    ▼
Subnet (public)
    │
    ▼ port 80 open
Security Group (devops-sg)
    │
    ▼
devops-ec2 (Nginx — accessible)
```

---

## Prerequisites

- AWS CLI configured with appropriate credentials
- An existing VPC named `devops-vpc`
- An existing EC2 instance named `devops-ec2` running Nginx
- An existing security group named `devops-sg`

---

## Root Cause

The VPC had an Internet Gateway that was **detached** from `devops-vpc` during an earlier configuration change. Without an attached IGW, the VPC has no path to the internet regardless of any other settings. Additionally, the route table had no `0.0.0.0/0` entry pointing to the IGW, meaning even after reattachment, traffic had no route to follow.

---

## Traffic Flow in a Public VPC

```
Internet
    │
    ▼  (1) Internet Gateway — the door between VPC and internet
    │      Must exist AND be attached to the VPC
    │
    ▼  (2) Route Table — 0.0.0.0/0 → igw-xxxxxxxx
    │      Must have a default route pointing to the IGW
    │
    ▼  (3) Subnet — must be public (auto-assign public IP enabled)
    │
    ▼  (4) Security Group — inbound port 80 open to 0.0.0.0/0
    │
    ▼  (5) EC2 Instance — must have a public IP address
    │
    ▼
Application (Nginx)
```

All five gates must be open. One closed gate anywhere in the chain makes the application unreachable — with no error message explaining why.

---

## Step-by-Step Troubleshooting and Fix

### 1. Gather Infrastructure Info

```bash
# Get the VPC
VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=tag:Name,Values=devops-vpc" \
  --query "Vpcs[0].VpcId" --output text)
echo "VPC ID: $VPC_ID"

# Get the EC2 instance
INSTANCE_ID=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=devops-ec2" \
            "Name=instance-state-name,Values=running" \
  --query "Reservations[0].Instances[0].InstanceId" --output text)
echo "Instance ID: $INSTANCE_ID"

# Get full instance details
aws ec2 describe-instances \
  --instance-ids $INSTANCE_ID \
  --query "Reservations[0].Instances[0].{PublicIP:PublicIpAddress,PrivateIP:PrivateIpAddress,Subnet:SubnetId,VPC:VpcId}" \
  --output table
```

### 2. Check and Fix Internet Gateway

```bash
# Check if an IGW is attached to the VPC
IGW_ID=$(aws ec2 describe-internet-gateways \
  --filters "Name=attachment.vpc-id,Values=$VPC_ID" \
  --query "InternetGateways[0].InternetGatewayId" --output text)
echo "IGW ID: $IGW_ID"

# If no IGW is attached, create and attach one
if [ "$IGW_ID" == "None" ] || [ -z "$IGW_ID" ]; then
  echo "No IGW found — creating one..."

  IGW_ID=$(aws ec2 create-internet-gateway \
    --query "InternetGateway.InternetGatewayId" --output text)
  echo "Created IGW: $IGW_ID"

  aws ec2 attach-internet-gateway \
    --internet-gateway-id $IGW_ID \
    --vpc-id $VPC_ID
  echo "IGW attached to VPC"
else
  echo "IGW already attached: $IGW_ID"
fi
```

### 3. Check and Fix Route Table

```bash
# Get the subnet the EC2 is in
SUBNET_ID=$(aws ec2 describe-instances \
  --instance-ids $INSTANCE_ID \
  --query "Reservations[0].Instances[0].SubnetId" --output text)
echo "Subnet ID: $SUBNET_ID"

# Get route table associated with that subnet
RT_ID=$(aws ec2 describe-route-tables \
  --filters "Name=association.subnet-id,Values=$SUBNET_ID" \
  --query "RouteTables[0].RouteTableId" --output text)

# Fall back to main route table if no explicit association
if [ "$RT_ID" == "None" ] || [ -z "$RT_ID" ]; then
  RT_ID=$(aws ec2 describe-route-tables \
    --filters "Name=vpc-id,Values=$VPC_ID" \
              "Name=association.main,Values=true" \
    --query "RouteTables[0].RouteTableId" --output text)
  echo "Using main route table: $RT_ID"
else
  echo "Route Table: $RT_ID"
fi

# Check current routes
aws ec2 describe-route-tables \
  --route-table-ids $RT_ID \
  --query "RouteTables[0].Routes[*].{Destination:DestinationCidrBlock,Target:GatewayId,State:State}" \
  --output table

# Add default route to IGW if missing
DEFAULT_ROUTE=$(aws ec2 describe-route-tables \
  --route-table-ids $RT_ID \
  --query "RouteTables[0].Routes[?DestinationCidrBlock=='0.0.0.0/0'].GatewayId" \
  --output text)

if [ -z "$DEFAULT_ROUTE" ] || [ "$DEFAULT_ROUTE" == "None" ]; then
  aws ec2 create-route \
    --route-table-id $RT_ID \
    --destination-cidr-block 0.0.0.0/0 \
    --gateway-id $IGW_ID
  echo "Default route added pointing to IGW"
else
  echo "Default route already exists: $DEFAULT_ROUTE"
fi
```

### 4. Ensure Subnet Auto-assigns Public IP

```bash
aws ec2 modify-subnet-attribute \
  --subnet-id $SUBNET_ID \
  --map-public-ip-on-launch

echo "Auto-assign public IP enabled on subnet"
```

### 5. Check EC2 Has a Public IP

```bash
PUBLIC_IP=$(aws ec2 describe-instances \
  --instance-ids $INSTANCE_ID \
  --query "Reservations[0].Instances[0].PublicIpAddress" --output text)
echo "EC2 Public IP: $PUBLIC_IP"

# If no public IP, allocate and associate an Elastic IP
if [ "$PUBLIC_IP" == "None" ] || [ -z "$PUBLIC_IP" ]; then
  echo "No public IP — allocating Elastic IP..."

  ALLOC_ID=$(aws ec2 allocate-address \
    --domain vpc \
    --query "AllocationId" --output text)
  echo "Elastic IP Allocation ID: $ALLOC_ID"

  aws ec2 associate-address \
    --instance-id $INSTANCE_ID \
    --allocation-id $ALLOC_ID
  echo "Elastic IP associated"

  PUBLIC_IP=$(aws ec2 describe-instances \
    --instance-ids $INSTANCE_ID \
    --query "Reservations[0].Instances[0].PublicIpAddress" --output text)
  echo "New Public IP: $PUBLIC_IP"
fi
```

### 6. Verify Security Group Rules

```bash
SG_ID=$(aws ec2 describe-security-groups \
  --filters "Name=group-name,Values=devops-sg" \
  --query "SecurityGroups[0].GroupId" --output text)
echo "SG ID: $SG_ID"

# Check current inbound rules
aws ec2 describe-security-groups \
  --group-ids $SG_ID \
  --query "SecurityGroups[0].IpPermissions[*].{From:FromPort,To:ToPort,Protocol:IpProtocol,CIDR:IpRanges[0].CidrIp}" \
  --output table

# Add port 80 if missing
aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0 2>/dev/null || echo "Port 80 rule already exists"
```

---

## Verification

```bash
# Test HTTP access
echo "Testing http://$PUBLIC_IP ..."
curl -s -o /dev/null -w "HTTP Status: %{http_code}\n" http://$PUBLIC_IP

echo "Access your app at: http://$PUBLIC_IP"
```

**Expected output:**
```
HTTP Status: 200
Access your app at: http://<public-ip>
```

Visit in browser:
```
http://<public-ip>
```
You should see the **Nginx welcome page** ✅

---

## Console Troubleshooting Steps

For those who prefer the AWS Console:

1. **VPC → Internet Gateways** — check IGW state is `Attached` to `devops-vpc`
2. **VPC → Route Tables** — filter by `devops-vpc`, check Routes tab for `0.0.0.0/0 → igw-xxx`
3. **VPC → Subnets** — check `Auto-assign public IPv4` is enabled
4. **EC2 → Instances** — check `Public IPv4 address` is not empty
5. **EC2 → Security Groups → devops-sg** — check inbound rule for port 80 from `0.0.0.0/0`

---

## Troubleshooting Checklist

| Check | Command | Expected Result |
|---|---|---|
| IGW attached | `describe-internet-gateways` | State: `attached` to VPC |
| Route to IGW | `describe-route-tables` | `0.0.0.0/0 → igw-xxxxxxxx` |
| Subnet public | `describe-subnets` | `MapPublicIpOnLaunch: true` |
| EC2 has public IP | `describe-instances` | `PublicIpAddress` not empty |
| SG port 80 open | `describe-security-groups` | Inbound TCP 80 from `0.0.0.0/0` |

---

## Common Mistakes

| Mistake | Consequence |
|---|---|
| IGW created but not attached to VPC | VPC still isolated — no internet path |
| IGW attached but no route table entry | Traffic has no direction — packets dropped |
| Route table updated but wrong subnet association | EC2 subnet uses different route table — still broken |
| Security group open but no public IP on EC2 | Internet has nowhere to send responses |
| Checking only security groups when debugging | Misses VPC-level issues entirely |

---

## Key Lesson

Security groups are the most visible access control layer in AWS, so they get checked first when something is unreachable. But security groups only control traffic **at the instance level**. If the VPC itself has no internet path, traffic never reaches the instance — and the security group never gets to make a decision.

Always troubleshoot in this order:

```
IGW → Route Table → Subnet → Public IP → Security Group → Application
```

Start at the network boundary and work inward. The fix will almost always be found in the first two steps.

---

## Technologies Used

- **AWS VPC** — Virtual Private Cloud networking
- **AWS Internet Gateway** — VPC internet connectivity
- **AWS Route Tables** — Traffic routing rules within VPC
- **AWS EC2** — Web server host running Nginx
- **AWS Security Groups** — Instance-level firewall rules
- **AWS Elastic IP** — Static public IP for EC2 instances
- **AWS CLI** — Diagnosis and fix via command line
