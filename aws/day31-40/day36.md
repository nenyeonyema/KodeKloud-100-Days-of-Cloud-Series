# AWS Task 36 — EC2 with Application Load Balancer (ALB) + Nginx

## Overview

This project provisions an EC2 instance running Nginx and places it behind an Application Load Balancer (ALB) on AWS. The setup demonstrates how to route internet traffic through an ALB to a backend EC2 instance using target groups, listeners, and properly scoped security groups — ensuring high availability and clean traffic management.

---

## Architecture

```
Internet
    │
    ▼
Application Load Balancer (xfusion-alb)
[Default Security Group — Port 80 open to 0.0.0.0/0]
    │
    ▼ HTTP Port 80
Target Group (xfusion-tg)
    │
    ▼ HTTP Port 80
EC2 Instance (xfusion-ec2)
[xfusion-sg — Port 80 open from Default SG only]
Nginx running via user data on launch
```

---

## Prerequisites

- AWS CLI configured with appropriate credentials
- IAM permissions for EC2, ELB, and VPC
- Default VPC available in `us-east-1`

---

## Resources Created

| Resource | Name | Details |
|---|---|---|
| Security Group | `xfusion-sg` | Allows port 80 from default SG only |
| EC2 Instance | `xfusion-ec2` | Ubuntu, t2.micro, Nginx via user data |
| Target Group | `xfusion-tg` | HTTP, port 80, instance type |
| Load Balancer | `xfusion-alb` | Internet-facing ALB, default SG |
| Listener | Port 80 | Forwards to `xfusion-tg` |

---

## Step-by-Step Setup

### 1. Gather VPC and Subnet Info

```bash
# Get default VPC
VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=isDefault,Values=true" \
  --query "Vpcs[0].VpcId" --output text)
echo "VPC: $VPC_ID"

# Get default security group
DEFAULT_SG=$(aws ec2 describe-security-groups \
  --filters "Name=group-name,Values=default" "Name=vpc-id,Values=$VPC_ID" \
  --query "SecurityGroups[0].GroupId" --output text)
echo "Default SG: $DEFAULT_SG"

# Get two subnets in different AZs for ALB
SUBNET1=$(aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query "Subnets[0].SubnetId" --output text)
SUBNET2=$(aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query "Subnets[1].SubnetId" --output text)
echo "Subnets: $SUBNET1, $SUBNET2"
```

### 2. Create xfusion-sg Security Group

```bash
XFUSION_SG=$(aws ec2 create-security-group \
  --group-name xfusion-sg \
  --description "Security group for xfusion EC2 - allows port 80 from default SG" \
  --vpc-id $VPC_ID \
  --query "GroupId" --output text)
echo "xfusion-sg: $XFUSION_SG"

# Allow port 80 only from the default SG (ALB -> EC2 traffic)
aws ec2 authorize-security-group-ingress \
  --group-id $XFUSION_SG \
  --protocol tcp \
  --port 80 \
  --source-group $DEFAULT_SG

echo "Port 80 from default SG added to xfusion-sg"
```

### 3. Open Port 80 on Default SG (Internet → ALB)

```bash
aws ec2 authorize-security-group-ingress \
  --group-id $DEFAULT_SG \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0

echo "Port 80 opened on default SG"
```

### 4. Get Latest Ubuntu AMI

```bash
AMI_ID=$(aws ec2 describe-images \
  --owners 099720109477 \
  --filters \
    "Name=name,Values=ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*" \
    "Name=state,Values=available" \
  --query "sort_by(Images, &CreationDate)[-1].ImageId" \
  --output text)
echo "Ubuntu AMI: $AMI_ID"
```

### 5. Launch EC2 Instance with Nginx User Data

```bash
cat > /tmp/userdata.sh << 'EOF'
#!/bin/bash
apt-get update -y
apt-get install -y nginx
systemctl start nginx
systemctl enable nginx
EOF

INSTANCE_ID=$(aws ec2 run-instances \
  --image-id $AMI_ID \
  --instance-type t2.micro \
  --security-group-ids $XFUSION_SG \
  --subnet-id $SUBNET1 \
  --user-data file:///tmp/userdata.sh \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=xfusion-ec2}]' \
  --query "Instances[0].InstanceId" --output text)
echo "Instance ID: $INSTANCE_ID"

# Wait for instance to be running
aws ec2 wait instance-running --instance-ids $INSTANCE_ID
echo "Instance is running"
```

> **Important:** Launch the EC2 instance in the same AZ/subnet that the ALB covers. A mismatch causes `Target.NotInUse` errors.

### 6. Create Target Group

```bash
TG_ARN=$(aws elbv2 create-target-group \
  --name xfusion-tg \
  --protocol HTTP \
  --port 80 \
  --vpc-id $VPC_ID \
  --target-type instance \
  --health-check-protocol HTTP \
  --health-check-path "/" \
  --query "TargetGroups[0].TargetGroupArn" --output text)
echo "Target Group ARN: $TG_ARN"
```

### 7. Register EC2 to Target Group

```bash
aws elbv2 register-targets \
  --target-group-arn $TG_ARN \
  --targets Id=$INSTANCE_ID

echo "EC2 registered to target group"
```

### 8. Create Application Load Balancer

```bash
ALB_ARN=$(aws elbv2 create-load-balancer \
  --name xfusion-alb \
  --subnets $SUBNET1 $SUBNET2 \
  --security-groups $DEFAULT_SG \
  --scheme internet-facing \
  --type application \
  --ip-address-type ipv4 \
  --query "LoadBalancers[0].LoadBalancerArn" --output text)
echo "ALB ARN: $ALB_ARN"

ALB_DNS=$(aws elbv2 describe-load-balancers \
  --load-balancer-arns $ALB_ARN \
  --query "LoadBalancers[0].DNSName" --output text)
echo "ALB DNS: $ALB_DNS"
```

### 9. Create Listener

```bash
aws elbv2 create-listener \
  --load-balancer-arn $ALB_ARN \
  --protocol HTTP \
  --port 80 \
  --default-actions Type=forward,TargetGroupArn=$TG_ARN

echo "Listener created — ALB forwards port 80 to xfusion-tg"
```

---

## Verification

```bash
# Wait for ALB to be active
aws elbv2 wait load-balancer-available --load-balancer-arns $ALB_ARN

# Check target health
aws elbv2 describe-target-health \
  --target-group-arn $TG_ARN \
  --query "TargetHealthDescriptions[*].{ID:Target.Id,Health:TargetHealth.State}" \
  --output table

# Test via curl
curl -s -o /dev/null -w "HTTP Status: %{http_code}\n" http://$ALB_DNS
```

**Expected output:**
```
| healthy | i-xxxxxxxxxxxxxxxxx |
HTTP Status: 200
```

Visit in browser:
```
http://<alb-dns-name>
```
You should see the **Nginx welcome page** ✅

---

## Security Design

```
Internet (0.0.0.0/0)
      │ Port 80
      ▼
 Default SG (on ALB)
      │ Port 80 — source: Default SG
      ▼
 xfusion-sg (on EC2)
```

- The EC2 instance is **not directly exposed** to the internet
- Port 80 on `xfusion-sg` only accepts traffic **from the ALB's security group**
- Direct access to the EC2 public IP on port 80 is blocked

---

## Troubleshooting

| Issue | Cause | Fix |
|---|---|---|
| `Target.NotInUse` | EC2 in AZ not covered by ALB | Relaunch EC2 in a subnet matching ALB's AZs |
| 503 Service Unavailable | Listener exists but no healthy targets | Re-register EC2 to target group |
| Connection refused on ALB DNS | Port 80 missing on default SG | Add inbound rule for port 80 → 0.0.0.0/0 |
| Health check failing | Nginx not started | Verify user data ran: `systemctl status nginx` |
| `InvalidPermission.Duplicate` | Security group rule already exists | Safe to ignore, rule is already in place |

---

## Key Concepts Demonstrated

- **ALB vs Direct EC2 access** — traffic flows through the load balancer, not directly to the instance
- **Security group chaining** — using a SG as a source instead of a CIDR for tighter access control
- **AZ alignment** — ALB subnets and EC2 subnet must be in matching Availability Zones
- **User data scripts** — automating software installation at instance launch with no manual SSH
- **Target group health checks** — ALB only routes to instances that pass HTTP health checks

---

## Technologies Used

- **AWS EC2** — Web server host (Ubuntu 22.04)
- **AWS ALB** — Application Load Balancer for traffic distribution
- **AWS Target Groups** — Backend pool with health checking
- **Nginx** — Web server installed via EC2 user data
- **AWS Security Groups** — Layered network access control
- **AWS CLI** — Full infrastructure provisioning
