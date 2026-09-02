# AWS Task 45 — NAT Gateway: Enabling Secure Outbound Internet Access for a Private EC2 Instance

## Overview

This project documents the setup of a NAT Gateway to provide outbound internet access to an EC2 instance running inside a private subnet. The instance had no public IP and no route to the internet, making it unable to communicate with external AWS services like S3. A public subnet was created in the same VPC, an Internet Gateway was attached, a NAT Gateway was deployed in the public subnet, and a dedicated private route table was explicitly associated with the private subnet — restoring outbound connectivity without exposing the instance to inbound internet traffic.

---

## Architecture

### Before Fix (Broken)
```
Internet
    │
    ✖ No path exists
    │
datacenter-priv-vpc (isolated)
    │
    └── datacenter-priv-subnet
              │
              └── datacenter-priv-ec2
                    (cron job failing silently —
                     no internet — S3 upload impossible)
```

### After Fix (Working)
```
Internet
    │
    ▼
Internet Gateway (datacenter-igw)
[attached to datacenter-priv-vpc]
    │
    ▼ route: 0.0.0.0/0 → igw
datacenter-pub-subnet (10.1.2.0/24)
    │
    ▼
NAT Gateway (datacenter-natgw)
[Elastic IP assigned]
    │
    ▼ route: 0.0.0.0/0 → natgw
datacenter-priv-rt
[explicitly associated with datacenter-priv-subnet]
    │
    ▼
datacenter-priv-subnet (10.1.1.0/24)
    │
    ▼
datacenter-priv-ec2
    │
    ▼ outbound only — no inbound exposure
S3 Bucket: datacenter-nat-30089
    └── datacenter-test.txt ✅ uploaded successfully
```

---

## Prerequisites

- AWS CLI configured with valid credentials
- IAM permissions for:
  - `ec2:CreateSubnet`
  - `ec2:CreateInternetGateway`
  - `ec2:AttachInternetGateway`
  - `ec2:CreateRouteTable`
  - `ec2:CreateRoute`
  - `ec2:AssociateRouteTable`
  - `ec2:AllocateAddress`
  - `ec2:CreateNatGateway`
- Existing resources already provisioned:
  - VPC: `datacenter-priv-vpc`
  - Private Subnet: `datacenter-priv-subnet` (`10.1.1.0/24`)
  - EC2 Instance: `datacenter-priv-ec2` (running in private subnet)
  - S3 Bucket: `datacenter-nat-30089`
- Region: `us-east-1`

---

## What Was Built

| Resource | Name | Configuration |
|---|---|---|
| Public Subnet | `datacenter-pub-subnet` | `10.1.2.0/24`, `us-east-1a`, public IP on launch |
| Internet Gateway | `datacenter-igw` | Attached to `datacenter-priv-vpc` |
| Public Route Table | `datacenter-pub-rt` | `0.0.0.0/0 → IGW`, associated with public subnet |
| Elastic IP | — | Allocated for NAT Gateway |
| NAT Gateway | `datacenter-natgw` | Deployed in public subnet with Elastic IP |
| Private Route Table | `datacenter-priv-rt` | `0.0.0.0/0 → NATGW`, explicitly associated with private subnet |

---

## Traffic Flow Explained

```
datacenter-priv-ec2 (no public IP)
        │
        │ outbound request (e.g. S3 upload)
        ▼
datacenter-priv-rt
        │ 0.0.0.0/0 → datacenter-natgw
        ▼
NAT Gateway (datacenter-natgw)
        │ translates private IP → Elastic IP
        ▼
Internet Gateway (datacenter-igw)
        │
        ▼
Internet / AWS S3
        │
        ▼ response comes back via same path
datacenter-priv-ec2 receives response ✅
```

The NAT Gateway performs **Network Address Translation** — it replaces the instance's private IP with its own Elastic IP for outbound traffic, then forwards responses back. The instance is never directly reachable from the internet.

---

## Step-by-Step Implementation

### 1. Verify Credentials and Get Existing Resources

```bash
aws sts get-caller-identity

VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=tag:Name,Values=datacenter-priv-vpc" \
  --query 'Vpcs[0].VpcId' \
  --output text --region us-east-1)
echo "VPC ID: $VPC_ID"

PRIV_SUBNET_ID=$(aws ec2 describe-subnets \
  --filters "Name=tag:Name,Values=datacenter-priv-subnet" \
  --query 'Subnets[0].SubnetId' \
  --output text --region us-east-1)
echo "Private Subnet ID: $PRIV_SUBNET_ID"

PRIV_AZ=$(aws ec2 describe-subnets \
  --subnet-ids $PRIV_SUBNET_ID \
  --query 'Subnets[0].AvailabilityZone' \
  --output text --region us-east-1)
echo "Private AZ: $PRIV_AZ"
```

### 2. Create Public Subnet

```bash
PUB_SUBNET_ID=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.1.2.0/24 \
  --availability-zone $PRIV_AZ \
  --query 'Subnet.SubnetId' \
  --output text --region us-east-1)
echo "Public Subnet ID: $PUB_SUBNET_ID"

aws ec2 create-tags \
  --resources $PUB_SUBNET_ID \
  --tags Key=Name,Value=datacenter-pub-subnet \
  --region us-east-1

aws ec2 modify-subnet-attribute \
  --subnet-id $PUB_SUBNET_ID \
  --map-public-ip-on-launch \
  --region us-east-1

echo "✅ Public subnet created and tagged"
```

### 3. Create and Attach Internet Gateway

```bash
IGW_ID=$(aws ec2 create-internet-gateway \
  --query 'InternetGateway.InternetGatewayId' \
  --output text --region us-east-1)
echo "IGW ID: $IGW_ID"

aws ec2 create-tags \
  --resources $IGW_ID \
  --tags Key=Name,Value=datacenter-igw \
  --region us-east-1

aws ec2 attach-internet-gateway \
  --internet-gateway-id $IGW_ID \
  --vpc-id $VPC_ID \
  --region us-east-1

echo "✅ Internet Gateway created and attached"
```

### 4. Create Public Route Table and Associate with Public Subnet

```bash
PUB_RT_ID=$(aws ec2 create-route-table \
  --vpc-id $VPC_ID \
  --query 'RouteTable.RouteTableId' \
  --output text --region us-east-1)
echo "Public RT ID: $PUB_RT_ID"

aws ec2 create-tags \
  --resources $PUB_RT_ID \
  --tags Key=Name,Value=datacenter-pub-rt \
  --region us-east-1

aws ec2 create-route \
  --route-table-id $PUB_RT_ID \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id $IGW_ID \
  --region us-east-1

aws ec2 associate-route-table \
  --route-table-id $PUB_RT_ID \
  --subnet-id $PUB_SUBNET_ID \
  --region us-east-1

echo "✅ Public route table created and associated with public subnet"
```

### 5. Allocate Elastic IP and Create NAT Gateway

```bash
EIP_ALLOC_ID=$(aws ec2 allocate-address \
  --domain vpc \
  --query 'AllocationId' \
  --output text --region us-east-1)
echo "EIP Allocation ID: $EIP_ALLOC_ID"

NATGW_ID=$(aws ec2 create-nat-gateway \
  --subnet-id $PUB_SUBNET_ID \
  --allocation-id $EIP_ALLOC_ID \
  --query 'NatGateway.NatGatewayId' \
  --output text --region us-east-1)
echo "NAT Gateway ID: $NATGW_ID"

aws ec2 create-tags \
  --resources $NATGW_ID \
  --tags Key=Name,Value=datacenter-natgw \
  --region us-east-1

echo "⏳ Waiting for NAT Gateway to become available..."
aws ec2 wait nat-gateway-available \
  --nat-gateway-ids $NATGW_ID \
  --region us-east-1

echo "✅ NAT Gateway is available!"
```

### 6. Create Private Route Table and Explicitly Associate with Private Subnet

```bash
PRIV_RT_ID=$(aws ec2 create-route-table \
  --vpc-id $VPC_ID \
  --query 'RouteTable.RouteTableId' \
  --output text --region us-east-1)
echo "Private RT ID: $PRIV_RT_ID"

aws ec2 create-tags \
  --resources $PRIV_RT_ID \
  --tags Key=Name,Value=datacenter-priv-rt \
  --region us-east-1

aws ec2 create-route \
  --route-table-id $PRIV_RT_ID \
  --destination-cidr-block 0.0.0.0/0 \
  --nat-gateway-id $NATGW_ID \
  --region us-east-1

# ⚠️ CRITICAL — explicitly associate with private subnet
aws ec2 associate-route-table \
  --route-table-id $PRIV_RT_ID \
  --subnet-id $PRIV_SUBNET_ID \
  --region us-east-1

echo "✅ Private route table created, NAT route added, explicitly associated with private subnet"
```

### 7. Wait and Verify S3 Upload

```bash
echo "⏳ Waiting 3 minutes for cron job to upload file..."
sleep 180

echo "=== S3 Bucket Contents ==="
aws s3 ls s3://datacenter-nat-30089/ --recursive --human-readable --region us-east-1
```

---

## Verification

**Confirm all resources are correctly configured:**
```bash
echo "=== Public Subnet ==="
aws ec2 describe-subnets --subnet-ids $PUB_SUBNET_ID \
  --query 'Subnets[0].{Name:Tags[?Key==`Name`].Value|[0],CIDR:CidrBlock,AZ:AvailabilityZone}' \
  --output table --region us-east-1

echo "=== Internet Gateway ==="
aws ec2 describe-internet-gateways --internet-gateway-ids $IGW_ID \
  --query 'InternetGateways[0].{Id:InternetGatewayId,State:Attachments[0].State}' \
  --output table --region us-east-1

echo "=== NAT Gateway ==="
aws ec2 describe-nat-gateways --nat-gateway-ids $NATGW_ID \
  --query 'NatGateways[0].{Name:Tags[?Key==`Name`].Value|[0],State:State,Subnet:SubnetId}' \
  --output table --region us-east-1

echo "=== Private Route Table ==="
aws ec2 describe-route-tables --route-table-ids $PRIV_RT_ID \
  --query 'RouteTables[0].{
    Routes:Routes[*].{Dest:DestinationCidrBlock,NAT:NatGatewayId,State:State},
    Associations:Associations[*].{Subnet:SubnetId,Main:Main}}' \
  --output json --region us-east-1
```

**Expected S3 output:**
```
2026-04-06 13:26:03    0 Bytes datacenter-test.txt  ✅
```

---

## Console Verification Steps

For those who prefer the AWS Console:

1. **VPC → Subnets** — confirm `datacenter-pub-subnet` exists with CIDR `10.1.2.0/24`
2. **VPC → Internet Gateways** — confirm `datacenter-igw` state is `Attached` to `datacenter-priv-vpc`
3. **VPC → NAT Gateways** — confirm `datacenter-natgw` state is `Available` in public subnet
4. **VPC → Route Tables → datacenter-pub-rt** — confirm route `0.0.0.0/0 → igw`
5. **VPC → Route Tables → datacenter-priv-rt** — confirm route `0.0.0.0/0 → natgw` and explicit subnet association with `datacenter-priv-subnet`
6. **S3 → datacenter-nat-30089** — confirm `datacenter-test.txt` is present

---

## Troubleshooting Checklist

| Check | Command | Expected Result |
|---|---|---|
| Public subnet exists | `describe-subnets` | CIDR `10.1.2.0/24`, AZ `us-east-1a` |
| IGW attached to VPC | `describe-internet-gateways` | State: `attached` |
| NAT Gateway available | `describe-nat-gateways` | State: `available` |
| Public RT has IGW route | `describe-route-tables` | `0.0.0.0/0 → igw-xxx` |
| Private RT has NAT route | `describe-route-tables` | `0.0.0.0/0 → nat-xxx` |
| Private subnet associated | `describe-route-tables` | `SubnetId` matches `datacenter-priv-subnet` |
| S3 file uploaded | `s3 ls` | `datacenter-test.txt` present |

---

## Common Mistakes

| Mistake | Consequence |
|---|---|
| Using wrong CIDR for public subnet | `InvalidSubnet.Range` error — must be within VPC CIDR and not overlap private subnet |
| Relying on main/implicit VPC route table for private subnet | Validation fails — private subnet must have an **explicit** route table association |
| Creating NAT Gateway in private subnet instead of public | NAT Gateway has no internet path — outbound traffic is still blocked |
| EIP allocation ID lost between commands | `InvalidElasticIpID.NotFound` — always re-allocate if session resets |
| Not waiting for NAT Gateway to reach `available` state | Route is added but traffic fails — NAT Gateway not ready to forward yet |
| Forgetting to tag resources | Validation scripts that look up resources by name tag will fail |

---

## Key Lesson

The most critical step in this task — and the one that caused the first attempt to fail — was **explicitly associating the private route table with the private subnet**. AWS subnets that have no explicit route table association fall back to the VPC's main route table implicitly. This implicit association is invisible in the console and undetectable without checking `describe-route-tables`. Validation scripts check for an explicit association and will fail if the subnet is only implicitly using the main route table.

Always follow this order when setting up NAT Gateway connectivity:

```
Get VPC & Private Subnet → Create Public Subnet → Create & Attach IGW
→ Create Public RT → Add IGW Route → Associate with Public Subnet
→ Allocate EIP → Create NAT Gateway in Public Subnet → Wait for Available
→ Create Private RT → Add NAT Route → Explicitly Associate with Private Subnet
→ Wait for Cron → Verify S3 Upload
```

Never assume a subnet is using the route table you intended — always confirm with `describe-route-tables` and check the `Associations` field explicitly.

---

## Technologies Used

- **AWS VPC** — Virtual Private Cloud with public and private subnet separation
- **AWS Internet Gateway** — Entry point for internet traffic into the VPC
- **AWS NAT Gateway** — Outbound-only internet access for private subnet resources
- **AWS Elastic IP** — Static public IP assigned to the NAT Gateway
- **AWS Route Tables** — Traffic routing rules with explicit subnet associations
- **AWS EC2** — Private instance running the upload cron job
- **AWS S3** — Target bucket for verifying outbound internet connectivity
- **AWS CLI** — Full infrastructure provisioning via command line
