# AWS Task 37 — EC2 to S3 Access via IAM Role

## Overview

This project configures an existing EC2 instance to securely access a private S3 bucket using an IAM Role — no hardcoded credentials, no access keys stored on the server. The setup demonstrates AWS best practices for granting EC2 instances permission to interact with S3 through least-privilege IAM policies and instance profiles, then validates the access by uploading and listing files from inside the instance.

---

## Architecture

```
aws-client host
      │
      │ SSH (key-based, passwordless)
      ▼
EC2 Instance (xfusion-ec2)
      │
      │ IAM Role (xfusion-role)
      │ Permissions: PutObject, GetObject, ListBucket
      ▼
S3 Bucket (xfusion-s3-26473)
[Private — no public access]
```

---

## Prerequisites

- AWS CLI configured with appropriate credentials
- An existing EC2 instance named `xfusion-ec2` in `us-east-1`
- `aws-client` host with SSH tools available

---

## Resources Created

| Resource | Name | Details |
|---|---|---|
| S3 Bucket | `xfusion-s3-26473` | Private, all public access blocked |
| IAM Policy | `xfusion-s3-policy` | PutObject, GetObject, ListBucket on bucket |
| IAM Role | `xfusion-role` | EC2 trust policy, policy attached |
| Instance Profile | `xfusion-role` | Wraps the IAM role for EC2 attachment |
| SSH Key | `/root/.ssh/id_rsa` | Generated on aws-client, added to EC2 root |

---

## Step-by-Step Setup

### 1. Get EC2 Instance Info

```bash
INSTANCE_ID=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=xfusion-ec2" \
            "Name=instance-state-name,Values=running" \
  --query "Reservations[0].Instances[0].InstanceId" \
  --output text)
echo "Instance ID: $INSTANCE_ID"

EC2_PUBLIC_IP=$(aws ec2 describe-instances \
  --instance-ids $INSTANCE_ID \
  --query "Reservations[0].Instances[0].PublicIpAddress" \
  --output text)
echo "Public IP: $EC2_PUBLIC_IP"

EC2_AZ=$(aws ec2 describe-instances \
  --instance-ids $INSTANCE_ID \
  --query "Reservations[0].Instances[0].Placement.AvailabilityZone" \
  --output text)
echo "AZ: $EC2_AZ"
```

### 2. Generate SSH Key and Add to EC2

```bash
# Generate SSH key if it doesn't exist
[ ! -f /root/.ssh/id_rsa ] && \
  ssh-keygen -t rsa -b 2048 -f /root/.ssh/id_rsa -N ""

# Send public key via EC2 Instance Connect (valid for 60 seconds)
aws ec2-instance-connect send-ssh-public-key \
  --instance-id $INSTANCE_ID \
  --instance-os-user ec2-user \
  --ssh-public-key file:///root/.ssh/id_rsa.pub \
  --availability-zone $EC2_AZ

# Copy key to root's authorized_keys for passwordless root SSH
ssh -o StrictHostKeyChecking=no -i /root/.ssh/id_rsa ec2-user@$EC2_PUBLIC_IP \
  "sudo mkdir -p /root/.ssh && \
   sudo cp ~/.ssh/authorized_keys /root/.ssh/authorized_keys && \
   sudo chmod 700 /root/.ssh && \
   sudo chmod 600 /root/.ssh/authorized_keys && \
   echo done"

# Verify root SSH access
ssh -o StrictHostKeyChecking=no -i /root/.ssh/id_rsa root@$EC2_PUBLIC_IP "whoami"
```

### 3. Create Private S3 Bucket

```bash
aws s3api create-bucket \
  --bucket xfusion-s3-26473 \
  --region us-east-1
echo "Bucket created"

# Block all public access
aws s3api put-public-access-block \
  --bucket xfusion-s3-26473 \
  --public-access-block-configuration \
  "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"
echo "Bucket is private"
```

### 4. Create IAM Policy

```bash
cat > /tmp/s3-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::xfusion-s3-26473",
        "arn:aws:s3:::xfusion-s3-26473/*"
      ]
    }
  ]
}
EOF

POLICY_ARN=$(aws iam create-policy \
  --policy-name xfusion-s3-policy \
  --policy-document file:///tmp/s3-policy.json \
  --query "Policy.Arn" --output text)
echo "Policy ARN: $POLICY_ARN"
```

### 5. Create IAM Role and Attach Policy

```bash
# Trust policy allowing EC2 to assume this role
cat > /tmp/trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {"Service": "ec2.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

# Create role
aws iam create-role \
  --role-name xfusion-role \
  --assume-role-policy-document file:///tmp/trust-policy.json
echo "Role created"

# Attach S3 policy to role
aws iam attach-role-policy \
  --role-name xfusion-role \
  --policy-arn $POLICY_ARN
echo "Policy attached"

# Create instance profile (bridge between EC2 and IAM role)
aws iam create-instance-profile \
  --instance-profile-name xfusion-role

# Add role to instance profile
aws iam add-role-to-instance-profile \
  --instance-profile-name xfusion-role \
  --role-name xfusion-role
echo "Instance profile ready"
```

### 6. Attach IAM Role to EC2 Instance

```bash
aws ec2 associate-iam-instance-profile \
  --instance-id $INSTANCE_ID \
  --iam-instance-profile Name=xfusion-role
echo "IAM role attached to EC2"

# Verify attachment
aws ec2 describe-iam-instance-profile-associations \
  --filters "Name=instance-id,Values=$INSTANCE_ID" \
  --query "IamInstanceProfileAssociations[0].{State:State,Profile:IamInstanceProfile.Arn}" \
  --output table
```

### 7. Test S3 Access from Inside EC2

```bash
# Allow role to propagate
sleep 15

ssh -o StrictHostKeyChecking=no -i /root/.ssh/id_rsa root@$EC2_PUBLIC_IP \
  "echo 'test content' > /tmp/testfile.txt && \
   aws s3 cp /tmp/testfile.txt s3://xfusion-s3-26473/ && \
   aws s3 ls s3://xfusion-s3-26473/"
```

---

## Verification

**Expected output:**
```
upload: /tmp/testfile.txt to s3://xfusion-s3-26473/testfile.txt
2026-03-11 15:XX:XX       13 testfile.txt
```

---

## How IAM Role Authentication Works

```
EC2 Instance
    │
    │ Requests temporary credentials
    ▼
Instance Metadata Service (169.254.169.254)
    │
    │ Returns short-lived Access Key + Secret + Token
    ▼
AWS STS (Security Token Service)
    │
    │ Validates against attached IAM Role
    ▼
S3 API — checks policy → allows or denies
```

- Credentials are **automatically rotated** every few hours by AWS
- No credentials are stored on the instance — ever
- The EC2 instance inherits permissions through the **Instance Profile**

---

## Security Highlights

- S3 bucket has **all public access blocked** — not accessible from the internet
- IAM policy follows **least privilege** — only the three required S3 actions are granted
- Permissions are scoped to the **specific bucket ARN only** — not all S3 buckets
- No AWS Access Keys or Secret Keys are stored on the EC2 instance
- SSH access uses **key-based authentication only** — no password login

---

## Troubleshooting

| Issue | Cause | Fix |
|---|---|---|
| `Permission denied (publickey)` | Public key not on EC2 | Re-run `send-ssh-public-key` — key is valid for 60s only |
| `Unable to locate credentials` | IAM role not yet propagated | Wait 15-30 seconds and retry |
| `Access Denied` on S3 | Policy not attached to role | Verify with `aws iam list-attached-role-policies --role-name xfusion-role` |
| SSH connect timeout | Port 22 not open on EC2 SG | Add inbound rule for TCP port 22 from 0.0.0.0/0 |
| `BucketAlreadyExists` | Bucket name taken globally | S3 bucket names are globally unique — choose a different name |

---

## Technologies Used

- **AWS EC2** — Application host
- **AWS S3** — Private object storage
- **AWS IAM** — Role, policy, and instance profile for secure access
- **AWS EC2 Instance Connect** — Passwordless SSH key bootstrapping
- **AWS STS** — Temporary credential vending via instance metadata
- **AWS CLI** — Full infrastructure provisioning and testing
