# Task 49 — Secure Log Aggregation with VPC Peering, EC2, and S3

## Overview

This task builds a secure and scalable log aggregation pipeline across two
AWS VPCs. Log files are gathered from a private EC2 instance running in an
isolated VPC, transferred securely to a public EC2 instance in a separate
VPC via VPC Peering, and then pushed to a private S3 bucket for long-term
storage. The entire transfer is automated using cron jobs on both instances.

---

## Architecture

```
┌─────────────────────────────────────┐
│         nautilus-priv-vpc           │
│         (existing VPC)              │
│                                     │
│  nautilus-priv-ec2                  │
│  /var/log/boots.log                 │
│         │                           │
│         │ cron (every min)          │
│         │ scp via VPC Peering       │
└─────────┼───────────────────────────┘
          │
          │ VPC Peering
          │ (nautilus-vpc-peering)
          │
┌─────────┼───────────────────────────┐
│         │   nautilus-pub-vpc        │
│         │   (new VPC)               │
│         ▼                           │
│  nautilus-pub-ec2                   │
│  /home/ubuntu/boots.log             │
│         │                           │
│         │ cron (every min)          │
│         │ aws s3 cp (via IAM role)  │
└─────────┼───────────────────────────┘
          │
          ▼
  S3 Bucket: nautilus-s3-logs-31924
  Path: nautilus-priv-vpc/boot/boots.log
```

---

## Resources

### Pre-existing (do not create)

| Resource | Name | Description |
|---|---|---|
| VPC | `nautilus-priv-vpc` | Private VPC — already exists |
| Subnet | `nautilus-priv-subnet` | Private subnet — already exists |
| Route Table | `nautilus-priv-rt` | Private route table — already exists |
| EC2 Instance | `nautilus-priv-ec2` | Ubuntu instance — already exists |
| Key Pair | `nautilus-key.pem` | At `/root/.ssh/` on aws-client |

### Created in this task

| Resource | Name | Description |
|---|---|---|
| VPC | `nautilus-pub-vpc` | New public VPC (10.1.0.0/16) |
| Subnet | `nautilus-pub-subnet` | Public subnet (10.1.1.0/24) |
| Route Table | `nautilus-pub-rt` | Public route table with IGW route |
| Internet Gateway | `nautilus-pub-igw` | Attached to public VPC |
| Security Group | `nautilus-pub-sg` | Controls access to public EC2 |
| EC2 Instance | `nautilus-pub-ec2` | Ubuntu, public subnet, IAM role attached |
| IAM Role | `nautilus-s3-role` | Grants S3 PutObject to public EC2 |
| S3 Bucket | `nautilus-s3-logs-31924` | Private bucket for log storage |
| VPC Peering | `nautilus-vpc-peering` | Connects private and public VPCs |

---

## Prerequisites

- AWS CLI configured with appropriate credentials
- Region: `us-east-1`
- SSH key at `/root/.ssh/nautilus-key.pem` on aws-client
- Private VPC and EC2 already exist

---

## Step-by-Step Setup

### Phase 1 — Gather existing info

```bash
# Set variables
MY_IP=$(curl -s ifconfig.me)
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
REGION="us-east-1"

# Get private VPC details
PRIV_VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=tag:Name,Values=nautilus-priv-vpc" \
  --query 'Vpcs[0].VpcId' --output text)

PRIV_VPC_CIDR=$(aws ec2 describe-vpcs \
  --vpc-ids $PRIV_VPC_ID \
  --query 'Vpcs[0].CidrBlock' --output text)

PRIV_EC2_ID=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=nautilus-priv-ec2" \
            "Name=instance-state-name,Values=running" \
  --query 'Reservations[0].Instances[0].InstanceId' --output text)

PRIV_EC2_IP=$(aws ec2 describe-instances \
  --instance-ids $PRIV_EC2_ID \
  --query 'Reservations[0].Instances[0].PrivateIpAddress' --output text)

[OPRIV_RT_ID=$(aws ec2 describe-route-tables \
  --filters "Name=tag:Name,Values=nautilus-priv-rt" \
  --query 'RouteTables[0].RouteTableId' --output text)

PRIV_SG_ID=$(aws ec2 describe-instances \
  --instance-ids $PRIV_EC2_ID \
  --query 'Reservations[0].Instances[0].SecurityGroups[0].GroupId' --output text)

echo "PRIV_VPC_ID  : $PRIV_VPC_ID"
echo "PRIV_VPC_CIDR: $PRIV_VPC_CIDR"
echo "PRIV_EC2_IP  : $PRIV_EC2_IP"
echo "PRIV_RT_ID   : $PRIV_RT_ID"
echo "PRIV_SG_ID   : $PRIV_SG_ID"
```

### Phase 2 — Verify SSH key

```bash
ls -la /root/.ssh/nautilus-key.pem
chmod 400 /root/.ssh/nautilus-key.pem
```

If key is missing, create a new one:
```bash
aws ec2 delete-key-pair --key-name nautilus-key 2>/dev/null || true
aws ec2 create-key-pair \
  --key-name nautilus-key \
  --query 'KeyMaterial' \
  --output text > /root/.ssh/nautilus-key.pem
chmod 400 /root/.ssh/nautilus-key.pem
```

### Phase 3 — Create public VPC

```bash
PUB_VPC_ID=$(aws ec2 create-vpc \
  --cidr-block 10.1.0.0/16 \
  --query 'Vpc.VpcId' --output text)

aws ec2 create-tags --resources $PUB_VPC_ID \
  --tags Key=Name,Value=nautilus-pub-vpc

aws ec2 modify-vpc-attribute \
  --vpc-id $PUB_VPC_ID --enable-dns-hostnames

aws ec2 modify-vpc-attribute \
  --vpc-id $PUB_VPC_ID --enable-dns-support

echo "Public VPC: $PUB_VPC_ID"
```

### Phase 4 — Create public subnet

```bash
PUB_SUBNET_ID=$(aws ec2 create-subnet \
  --vpc-id $PUB_VPC_ID \
  --cidr-block 10.1.1.0/24 \
  --availability-zone us-east-1a \
  --query 'Subnet.SubnetId' --output text)

aws ec2 create-tags --resources $PUB_SUBNET_ID \
  --tags Key=Name,Value=nautilus-pub-subnet

aws ec2 modify-subnet-attribute \
  --subnet-id $PUB_SUBNET_ID --map-public-ip-on-launch

echo "Public Subnet: $PUB_SUBNET_ID"
```

### Phase 5 — Create and attach Internet Gateway

```bash
IGW_ID=$(aws ec2 create-internet-gateway \
  --query 'InternetGateway.InternetGatewayId' --output text)

aws ec2 create-tags --resources $IGW_ID \
  --tags Key=Name,Value=nautilus-pub-igw

aws ec2 attach-internet-gateway \
  --internet-gateway-id $IGW_ID \
  --vpc-id $PUB_VPC_ID

echo "IGW: $IGW_ID"
```

### Phase 6 — Create public route table

```bash
PUB_RT_ID=$(aws ec2 create-route-table \
  --vpc-id $PUB_VPC_ID \
  --query 'RouteTable.RouteTableId' --output text)

aws ec2 create-tags --resources $PUB_RT_ID \
  --tags Key=Name,Value=nautilus-pub-rt

aws ec2 create-route \
  --route-table-id $PUB_RT_ID \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id $IGW_ID

aws ec2 associate-route-table \
  --route-table-id $PUB_RT_ID \
  --subnet-id $PUB_SUBNET_ID

echo "Public RT: $PUB_RT_ID"
```

### Phase 7 — Create security groups

```bash
# Create public SG
PUB_SG_ID=$(aws ec2 create-security-group \
  --group-name nautilus-pub-sg \
  --description "nautilus-pub-ec2 SG" \
  --vpc-id $PUB_VPC_ID \
  --query 'GroupId' --output text)

aws ec2 create-tags --resources $PUB_SG_ID \
  --tags Key=Name,Value=nautilus-pub-sg

# Allow SSH from aws-client IP
aws ec2 authorize-security-group-ingress \
  --group-id $PUB_SG_ID --protocol tcp --port 22 \
  --cidr ${MY_IP}/32

# Allow SSH from anywhere as backup
aws ec2 authorize-security-group-ingress \
  --group-id $PUB_SG_ID --protocol tcp --port 22 \
  --cidr 0.0.0.0/0

# Allow all traffic from private VPC
aws ec2 authorize-security-group-ingress \
  --group-id $PUB_SG_ID --protocol all --port -1 \
  --cidr $PRIV_VPC_CIDR

# Update private SG to allow traffic from public VPC
aws ec2 authorize-security-group-ingress \
  --group-id $PRIV_SG_ID --protocol all --port -1 \
  --cidr 10.1.0.0/16

echo "Public SG: $PUB_SG_ID"
```

### Phase 8 — Create IAM role

```bash
# Create role
aws iam create-role \
  --role-name nautilus-s3-role \
  --assume-role-policy-document '{
    "Version":"2012-10-17",
    "Statement":[{
      "Effect":"Allow",
      "Principal":{"Service":"ec2.amazonaws.com"},
      "Action":"sts:AssumeRole"
    }]
  }'

# Create policy — use dynamic account ID, never hardcode
POLICY_ARN=$(aws iam create-policy \
  --policy-name nautilus-s3-put-policy \
  --policy-document "{
    \"Version\":\"2012-10-17\",
    \"Statement\":[{
      \"Effect\":\"Allow\",
      \"Action\":[\"s3:PutObject\",\"s3:GetObject\",\"s3:ListBucket\"],
      \"Resource\":[
        \"arn:aws:s3:::nautilus-s3-logs-31924\",
        \"arn:aws:s3:::nautilus-s3-logs-31924/*\"
      ]
    }]
  }" \
  --query 'Policy.Arn' --output text)

echo "Policy ARN: $POLICY_ARN"

# Attach policy to role
aws iam attach-role-policy \
  --role-name nautilus-s3-role \
  --policy-arn $POLICY_ARN

# Create and configure instance profile
aws iam create-instance-profile \
  --instance-profile-name nautilus-s3-role

aws iam add-role-to-instance-profile \
  --instance-profile-name nautilus-s3-role \
  --role-name nautilus-s3-role

echo "Waiting 15s for IAM propagation..."
sleep 15
```

### Phase 9 — Launch public EC2

```bash
# Get latest Ubuntu 22.04 AMI
AMI_ID=$(aws ec2 describe-images \
  --owners amazon \
  --filters "Name=name,Values=ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*" \
            "Name=state,Values=available" \
  --query 'sort_by(Images,&CreationDate)[-1].ImageId' \
  --output text)

echo "AMI: $AMI_ID"

# Launch instance
PUB_EC2_ID=$(aws ec2 run-instances \
  --image-id $AMI_ID \
  --instance-type t2.micro \
  --key-name nautilus-key \
  --subnet-id $PUB_SUBNET_ID \
  --security-group-ids $PUB_SG_ID \
  --iam-instance-profile Name=nautilus-s3-role \
  --associate-public-ip-address \
  --query 'Instances[0].InstanceId' --output text)

aws ec2 create-tags --resources $PUB_EC2_ID \
  --tags Key=Name,Value=nautilus-pub-ec2

aws ec2 wait instance-running --instance-ids $PUB_EC2_ID

# Get IPs and AZ — always fetch dynamically
PUB_EC2_PUBLIC_IP=$(aws ec2 describe-instances \
  --instance-ids $PUB_EC2_ID \
  --query 'Reservations[0].Instances[0].PublicIpAddress' --output text)

PUB_EC2_PRIV_IP=$(aws ec2 describe-instances \
  --instance-ids $PUB_EC2_ID \
  --query 'Reservations[0].Instances[0].PrivateIpAddress' --output text)

PUB_EC2_AZ=$(aws ec2 describe-instances \
  --instance-ids $PUB_EC2_ID \
  --query 'Reservations[0].Instances[0].Placement.AvailabilityZone' --output text)

echo "Public IP : $PUB_EC2_PUBLIC_IP"
echo "Private IP: $PUB_EC2_PRIV_IP"
echo "AZ        : $PUB_EC2_AZ"
```

### Phase 10 — Create private S3 bucket

```bash
aws s3api create-bucket \
  --bucket nautilus-s3-logs-31924 \
  --region us-east-1

aws s3api put-public-access-block \
  --bucket nautilus-s3-logs-31924 \
  --public-access-block-configuration \
  "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"

echo "S3 bucket ready."
```

### Phase 11 — VPC Peering and route updates

```bash
# Create peering connection
PEERING_ID=$(aws ec2 create-vpc-peering-connection \
  --vpc-id $PUB_VPC_ID \
  --peer-vpc-id $PRIV_VPC_ID \
  --query 'VpcPeeringConnection.VpcPeeringConnectionId' --output text)

aws ec2 create-tags --resources $PEERING_ID \
  --tags Key=Name,Value=nautilus-vpc-peering

# Accept the peering connection
aws ec2 accept-vpc-peering-connection \
  --vpc-peering-connection-id $PEERING_ID

sleep 5

# Route public → private via peering
aws ec2 create-route \
  --route-table-id $PUB_RT_ID \
  --destination-cidr-block $PRIV_VPC_CIDR \
  --vpc-peering-connection-id $PEERING_ID

# Route private → public via peering
aws ec2 create-route \
  --route-table-id $PRIV_RT_ID \
  --destination-cidr-block 10.1.0.0/16 \
  --vpc-peering-connection-id $PEERING_ID

echo "Peering: $PEERING_ID — routes updated on both sides."
```

### Phase 12 — SSH into public EC2

```bash
# Use EC2 Instance Connect API — works even when key is missing
# Always fetch AZ dynamically, never hardcode
PUB_KEY=$(ssh-keygen -y -f /root/.ssh/nautilus-key.pem)

aws ec2-instance-connect send-ssh-public-key \
  --instance-id $PUB_EC2_ID \
  --instance-os-user ubuntu \
  --ssh-public-key "$PUB_KEY" \
  --availability-zone $PUB_EC2_AZ \
  --region us-east-1 && \
ssh -i /root/.ssh/nautilus-key.pem \
  -o StrictHostKeyChecking=no \
  ubuntu@$PUB_EC2_PUBLIC_IP
```

### Phase 13 — Configure public EC2

Once inside the public EC2:

```bash
# Install AWS CLI v2
sudo apt-get update -qq
sudo apt-get install -y unzip curl
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" \
  -o /tmp/awscliv2.zip
unzip -q /tmp/awscliv2.zip -d /tmp
sudo /tmp/aws/install
aws --version

# Confirm IAM role is attached
aws sts get-caller-identity

# Set cron job
crontab -e
```

Add this line in the cron editor:
```
* * * * * /usr/local/bin/aws s3 cp /home/ubuntu/boots.log s3://nautilus-s3-logs-31924/nautilus-priv-vpc/boot/boots.log --region us-east-1
```

Confirm and exit:
```bash
crontab -l
exit
```

### Phase 14 — Copy key to public EC2

From aws-client:
```bash
scp -i /root/.ssh/nautilus-key.pem \
  -o StrictHostKeyChecking=no \
  /root/.ssh/nautilus-key.pem \
  ubuntu@$PUB_EC2_PUBLIC_IP:/home/ubuntu/nautilus-key.pem
```

### Phase 15 — SSH into private EC2

```bash
# SSH into public EC2 first, then hop to private
PUB_KEY=$(ssh-keygen -y -f /root/.ssh/nautilus-key.pem)

aws ec2-instance-connect send-ssh-public-key \
  --instance-id $PUB_EC2_ID \
  --instance-os-user ubuntu \
  --ssh-public-key "$PUB_KEY" \
  --availability-zone $PUB_EC2_AZ \
  --region us-east-1 && \
ssh -i /root/.ssh/nautilus-key.pem \
  -o StrictHostKeyChecking=no \
  ubuntu@$PUB_EC2_PUBLIC_IP
```

Then from inside the public EC2:
```bash
ssh -i /home/ubuntu/nautilus-key.pem \
  -o StrictHostKeyChecking=no \
  ubuntu@<PRIV_EC2_IP>
```

### Phase 16 — Configure private EC2

From public EC2, copy the key to private EC2 first:
```bash
scp -i /home/ubuntu/nautilus-key.pem \
  -o StrictHostKeyChecking=no \
  /home/ubuntu/nautilus-key.pem \
  ubuntu@<PRIV_EC2_IP>:/home/ubuntu/nautilus-key.pem
```

Once inside the private EC2:
```bash
chmod 600 /home/ubuntu/nautilus-key.pem

# Confirm boots.log exists
ls -la /var/log/boots.log

# If missing, create it
echo "$(date) boot log" | sudo tee /var/log/boots.log

# Set cron job
crontab -e
```

Add this line (replace with actual public EC2 private IP):
```
* * * * * scp -i /home/ubuntu/nautilus-key.pem -o StrictHostKeyChecking=no /var/log/boots.log ubuntu@10.1.1.x:/home/ubuntu/boots.log
```

Confirm and exit:
```bash
crontab -l
exit
exit
```

---

## Verification

Wait 2 minutes for cron to run, then verify from aws-client:

```bash
# Main check — file in S3
aws s3 ls s3://nautilus-s3-logs-31924/nautilus-priv-vpc/boot/

# Check boots.log arrived on public EC2
ssh -i /root/.ssh/nautilus-key.pem \
  ubuntu@$PUB_EC2_PUBLIC_IP \
  "ls -la /home/ubuntu/boots.log 2>/dev/null || echo NOT FOUND"

# Check cron on public EC2
ssh -i /root/.ssh/nautilus-key.pem \
  ubuntu@$PUB_EC2_PUBLIC_IP "crontab -l"
```

---

## Log File Path in S3

```
s3://nautilus-s3-logs-31924/nautilus-priv-vpc/boot/boots.log
```

---

## Cron Job Summary

| Instance | Cron Schedule | Command |
|---|---|---|
| `nautilus-priv-ec2` | Every minute | `scp /var/log/boots.log → public EC2:/home/ubuntu/boots.log` |
| `nautilus-pub-ec2` | Every minute | `aws s3 cp /home/ubuntu/boots.log → S3 bucket` |

---

## IAM Permissions Summary

| Permission | Resource | Purpose |
|---|---|---|
| `s3:PutObject` | `nautilus-s3-logs-31924/*` | Upload log file to S3 |
| `s3:GetObject` | `nautilus-s3-logs-31924/*` | Read objects from bucket |
| `s3:ListBucket` | `nautilus-s3-logs-31924` | List bucket contents |

---

## Key Lessons Learned

| Mistake | Root Cause | Fix |
|---|---|---|
| `Permission denied (publickey)` | Wrong SSH user | Always use `ubuntu` for Ubuntu AMI, `ec2-user` for Amazon Linux |
| SSH connection timeout | Port 22 blocked in SG | Add `MY_IP/32` to SG at creation time, not after |
| Instance Connect wrong AZ | AZ hardcoded as `us-east-1a` | Always fetch AZ dynamically with `describe-instances` |
| Policy attach failed | Hardcoded wrong account ID in ARN | Always capture ARN from `create-policy` output directly |
| ProxyJump failing | Public EC2 rejecting jump | SSH into public EC2 first, then SSH to private from there |
| `scp: Permission denied` | Destination was `/root/` (root-owned) | Use `/home/ubuntu/` as destination instead |
| boots.log missing | File does not exist on private EC2 | Create with `sudo tee /var/log/boots.log` |
[I| Variable empty on remote EC2 | Shell variables don't carry over SSH | Always use actual IP values inside remote sessions |

---

## Networking Summary

### VPC Peering Route Table Updates

| Route Table | Destination | Target |
|---|---|---|
| `nautilus-pub-rt` | `<priv-vpc-cidr>` | `nautilus-vpc-peering` |
| `nautilus-priv-rt` | `10.1.0.0/16` | `nautilus-vpc-peering` |
| `nautilus-pub-rt` | `0.0.0.0/0` | Internet Gateway |

---

## Cleanup

```bash
# Remove cron jobs (on each EC2)
crontab -r

# Delete S3 objects and bucket
aws s3 rm s3://nautilus-s3-logs-31924 --recursive
aws s3api delete-bucket --bucket nautilus-s3-logs-31924

# Delete peering connection
aws ec2 delete-vpc-peering-connection \
  --vpc-peering-connection-id $PEERING_ID

# Terminate public EC2
aws ec2 terminate-instances --instance-ids $PUB_EC2_ID
aws ec2 wait instance-terminated --instance-ids $PUB_EC2_ID

# Delete public VPC resources
aws ec2 delete-subnet --subnet-id $PUB_SUBNET_ID
aws ec2 delete-route-table --route-table-id $PUB_RT_ID
aws ec2 detach-internet-gateway \
  --internet-gateway-id $IGW_ID --vpc-id $PUB_VPC_ID
aws ec2 delete-internet-gateway --internet-gateway-id $IGW_ID
aws ec2 delete-security-group --group-id $PUB_SG_ID
aws ec2 delete-vpc --vpc-id $PUB_VPC_ID
[O
# Delete IAM resources
aws iam remove-role-from-instance-profile \
  --instance-profile-name nautilus-s3-role \
  --role-name nautilus-s3-role
aws iam delete-instance-profile \
  --instance-profile-name nautilus-s3-role
aws iam detach-role-policy \
  --role-name nautilus-s3-role \
  --policy-arn $POLICY_ARN
aws iam delete-role --role-name nautilus-s3-role
aws iam delete-policy --policy-arn $POLICY_ARN
```

---

## Verification Checklist

- [ ] Private VPC and EC2 exist and are running
- [ ] Public VPC created with correct CIDR `10.1.0.0/16`
- [ ] Public subnet has auto-assign public IP enabled
- [ ] Internet Gateway attached and route `0.0.0.0/0` in public RT
- [ ] VPC Peering status is `active`
- [ ] Both route tables updated with peering routes
- [ ] IAM role attached to public EC2 instance
- [ ] AWS CLI v2 installed on public EC2
- [ ] `aws sts get-caller-identity` returns `nautilus-s3-role` on public EC2
- [ ] Cron job set on private EC2 with correct public EC2 private IP
- [ ] Cron job set on public EC2 with correct S3 path
- [ ] `boots.log` exists on private EC2 at `/var/log/boots.log`
- [ ] After 2 minutes: `aws s3 ls s3://nautilus-s3-logs-31924/nautilus-priv-vpc/boot/` shows `boots.log`

---

## Author
Nenye — Cloud & DevOps Engineer
Stack: AWS VPC · EC2 · S3 · IAM · VPC Peering · Cron
Region: us-east-1
