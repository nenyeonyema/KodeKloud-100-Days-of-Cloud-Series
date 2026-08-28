# AWS Task 35 — Private RDS Instance with EC2 Connectivity

## Overview

This project provisions a private MySQL RDS instance on AWS and connects it to an existing EC2 instance running a PHP web application. The setup demonstrates secure database connectivity using security group rules, passwordless SSH access via EC2 Instance Connect, and a PHP application that validates the database connection through a browser.

---

## Architecture

```
Internet
    │
    ▼
EC2 (nautilus-ec2)  ──── Port 3306 ────▶  RDS MySQL (nautilus-rds)
    │                                        (Private, no public access)
    │
Port 80 (HTTP)
    │
    ▼
Browser → "Connected successfully"
```

---

## Prerequisites

- AWS CLI configured with appropriate credentials
- An existing EC2 instance named `nautilus-ec2`
- SSH key pair available on the `aws-client` host
- `index.php` file located at `/root/index.php` on the `aws-client` host

---

## Resources Created

| Resource | Name | Details |
|---|---|---|
| RDS Instance | `nautilus-rds` | MySQL 8.4.5, db.t3.micro, 5GiB gp2 |
| Database | `nautilus_db` | Created during RDS provisioning |
| DB Subnet Group | `nautilus-db-subnet-group` | Spans 2 AZs in the VPC |
| Security Group | `nautilus-rds-sg` | Allows port 3306 from EC2 SG only |
| EC2 Port Rule | Port 80 | Added to EC2 security group |

---

## Step-by-Step Setup

### 1. Gather Existing Infrastructure Info

```bash
EC2_ID=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=nautilus-ec2" \
  --query "Reservations[0].Instances[0].InstanceId" \
  --output text)

VPC_ID=$(aws ec2 describe-instances \
  --instance-ids $EC2_ID \
  --query "Reservations[0].Instances[0].VpcId" \
  --output text)

EC2_SG=$(aws ec2 describe-instances \
  --instance-ids $EC2_ID \
  --query "Reservations[0].Instances[0].SecurityGroups[0].GroupId" \
  --output text)
```

### 2. Create RDS Security Group

```bash
RDS_SG=$(aws ec2 create-security-group \
  --group-name nautilus-rds-sg \
  --description "Security group for nautilus RDS" \
  --vpc-id $VPC_ID \
  --query "GroupId" --output text)

# Allow MySQL port only from EC2 security group
aws ec2 authorize-security-group-ingress \
  --group-id $RDS_SG \
  --protocol tcp \
  --port 3306 \
  --source-group $EC2_SG
```

### 3. Open Port 80 on EC2 Security Group

```bash
aws ec2 authorize-security-group-ingress \
  --group-id $EC2_SG \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0
```

### 4. Create DB Subnet Group

```bash
SUBNET1=$(aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query "Subnets[0].SubnetId" --output text)

SUBNET2=$(aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query "Subnets[1].SubnetId" --output text)

aws rds create-db-subnet-group \
  --db-subnet-group-name nautilus-db-subnet-group \
  --db-subnet-group-description "Subnet group for nautilus RDS" \
  --subnet-ids $SUBNET1 $SUBNET2
```

### 5. Create RDS Instance

```bash
aws rds create-db-instance \
  --db-instance-identifier nautilus-rds \
  --db-instance-class db.t3.micro \
  --engine mysql \
  --engine-version 8.4.5 \
  --master-username nautilus_admin \
  --master-user-password "Admin1234!" \
  --allocated-storage 5 \
  --storage-type gp2 \
  --db-name nautilus_db \
  --db-subnet-group-name nautilus-db-subnet-group \
  --vpc-security-group-ids $RDS_SG \
  --no-publicly-accessible \
  --no-multi-az \
  --no-deletion-protection

# Wait for RDS to be available (~5-10 minutes)
aws rds wait db-instance-available \
  --db-instance-identifier nautilus-rds

RDS_ENDPOINT=$(aws rds describe-db-instances \
  --db-instance-identifier nautilus-rds \
  --query "DBInstances[0].Endpoint.Address" \
  --output text)
```

### 6. Setup Passwordless SSH to EC2

```bash
# Generate SSH key if it doesn't exist
[ ! -f /root/.ssh/id_rsa ] && ssh-keygen -t rsa -b 2048 -f /root/.ssh/id_rsa -N ""

EC2_AZ=$(aws ec2 describe-instances \
  --instance-ids $EC2_ID \
  --query "Reservations[0].Instances[0].Placement.AvailabilityZone" \
  --output text)

# Send public key via EC2 Instance Connect
aws ec2-instance-connect send-ssh-public-key \
  --instance-id $EC2_ID \
  --instance-os-user ec2-user \
  --ssh-public-key file:///root/.ssh/id_rsa.pub \
  --availability-zone $EC2_AZ

# Add key to root's authorized_keys
ssh -o StrictHostKeyChecking=no -i /root/.ssh/id_rsa ec2-user@$EC2_PUBLIC_IP \
  "sudo mkdir -p /root/.ssh && \
   sudo cp ~/.ssh/authorized_keys /root/.ssh/authorized_keys && \
   sudo chmod 700 /root/.ssh && \
   sudo chmod 600 /root/.ssh/authorized_keys"
```

### 7. Configure and Deploy index.php

```bash
# Update index.php with RDS connection details
cat > /root/index.php << 'EOF'
<?php
$servername = "RDS_ENDPOINT";
$username   = "nautilus_admin";
$password   = "Admin1234!";
$dbname     = "nautilus_db";

$conn = new mysqli($servername, $username, $password, $dbname);

if ($conn->connect_error) {
    die("Connection failed: " . $conn->connect_error);
}
echo "Connected successfully";
$conn->close();
?>
EOF

sed -i "s|RDS_ENDPOINT|$RDS_ENDPOINT|g" /root/index.php

# Install Apache and PHP on EC2
ssh -o StrictHostKeyChecking=no -i /root/.ssh/id_rsa root@$EC2_PUBLIC_IP \
  "yum install -y httpd php php-mysqli && systemctl start httpd && systemctl enable httpd"

# Copy index.php to EC2
scp -o StrictHostKeyChecking=no -i /root/.ssh/id_rsa \
  /root/index.php root@$EC2_PUBLIC_IP:/var/www/html/
```

---

## Verification

```bash
# Test via curl
curl http://$EC2_PUBLIC_IP/index.php
```

**Expected output:**
```
Connected successfully
```

---

## Security Highlights

- RDS instance has **no public access** — only reachable within the VPC
- MySQL port 3306 is restricted to the **EC2 security group only** — no open CIDR ranges
- Port 80 on EC2 is open to the internet for web traffic only
- SSH access uses **key-based authentication** — no password login

---

---

## Technologies Used

- **AWS RDS** — Managed MySQL 8.4.5 database
- **AWS EC2** — Web server host
- **AWS EC2 Instance Connect** — Passwordless SSH bootstrapping
- **Apache HTTP Server** — Web server on EC2
- **PHP + mysqli** — Database connectivity
- **AWS CLI** — Full infrastructure provisioning
