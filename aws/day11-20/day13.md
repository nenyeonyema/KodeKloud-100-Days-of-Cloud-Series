# AWS Task 13 — Create an AMI from an EC2 Instance

## 📌 Task Overview

In this task, the Nautilus DevOps team required a reusable machine image created from an existing EC2 instance. The goal was to build an Amazon Machine Image (AMI) named `devops-ec2-ami` from the `devops-ec2` instance in `us-east-1` and ensure the image reached the `available` state before completion. The entire operation was performed and verified using the AWS CLI.

| Requirement | Value |
| :--- | :--- |
| **EC2 Instance** | `devops-ec2` |
| **AMI Name** | `devops-ec2-ami` |
| **AWS Region** | `us-east-1` |
| **Target AMI State** | `available` |
| **Method** | AWS CLI |

---

## 🎯 Objective

Generate a reusable AMI template backed by underlying EBS volume snapshots:

```text
EC2 Instance (devops-ec2) ──► create-image ──► EBS Snapshots ──► AMI (devops-ec2-ami: available)
```

---

## 🛠️ Prerequisites

Set the active region for the AWS CLI session:
```bash
export AWS_DEFAULT_REGION=us-east-1
aws configure get region
```

**Expected output:**
```text
us-east-1
```

---

## 1. Retrieve and Verify the EC2 Instance

Locate the instance ID using the `Name` tag filter:
```bash
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=devops-ec2" \
  --query "Reservations[].Instances[].InstanceId" \
  --output text
```

**Example output:**
```text
i-0123456789abcdef0
```

Save the instance ID:
```bash
INSTANCE_ID=i-0123456789abcdef0
```

Verify that the instance is in the `running` state:
```bash
aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query "Reservations[].Instances[].State.Name" \
  --output text
```

> **Note:** If the instance is stopped, start it and wait until it transitions to `running`:
> ```bash
> aws ec2 start-instances --instance-ids "$INSTANCE_ID"
> aws ec2 wait instance-running --instance-ids "$INSTANCE_ID"
> ```

---

## 2. Create the Amazon Machine Image (AMI)

Execute the image creation command against the instance:

```bash
aws ec2 create-image \
  --instance-id "$INSTANCE_ID" \
  --name "devops-ec2-ami" \
  --description "AMI created from devops-ec2"
```

**Example JSON response:**
```json
{
    "ImageId": "ami-0123456789abcdef0"
}
```

Save the returned Image ID:
```bash
IMAGE_ID=ami-0123456789abcdef0
```

---

## 3. Monitor and Wait for AMI Availability

Immediately after creation, the AMI enters a `pending` state while AWS creates snapshot backups of attached EBS volumes:

```bash
aws ec2 describe-images \
  --image-ids "$IMAGE_ID" \
  --query "Images[].{Name:Name,ImageId:ImageId,State:State}" \
  --output table
```

Block execution until the snapshot creation finishes and the AMI becomes `available`:
```bash
aws ec2 wait image-available \
  --image-ids "$IMAGE_ID"
```

---

## 4. Verify AMI Creation

Confirm that the image reflects the `available` state:

```bash
aws ec2 describe-images \
  --image-ids "$IMAGE_ID" \
  --query "Images[].{ImageId:ImageId,Name:Name,State:State}" \
  --output table
```

**Expected output:**
```text
----------------------------------------------------
|                  DescribeImages                  |
+------------------+-------------------------------+
| ImageId          | ami-0123456789abcdef0         |
| Name             | devops-ec2-ami                |
| State            | available                     |
+------------------+-------------------------------+
```

---

## 🔍 Complete Verification

Verify the AMI configuration by searching owned images by name filter:

```bash
aws ec2 describe-images \
  --owners self \
  --filters "Name=name,Values=devops-ec2-ami" \
  --query "Images[].{ImageId:ImageId,Name:Name,State:State}" \
  --output table
```

---

## 🧠 Key Concepts & Takeaways

| Concept | Description |
| :--- | :--- |
| **AMI Architecture** | An AMI serves as a master blueprint containing block device mappings, OS configurations, installed software, and system files. |
| **Asynchronous Creation** | `create-image` returns immediately with an `ImageId`, but the image remains in `pending` until storage snapshots complete. |
| **EBS Snapshots Dependency** | Creating an AMI automatically triggers backing EBS volume snapshots under the hood. |
| **Golden Images** | Pre-configured AMIs accelerate Auto Scaling instance launch times and eliminate repetitive post-launch bootstrap scripts. |
| **Storage Overhead** | Retaining unused AMIs incurs costs for underlying EBS snapshots; old images and snapshots should be deregistered per retention policies. |

> *This was Task 13 of 50 AWS tasks in the AWS phase of my KodeKloud 100 Days of Cloud Challenge.*
