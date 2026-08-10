# AWS Task 5 — Create an EBS Volume

## 📌 Task Overview

As part of the Nautilus DevOps team's AWS migration, a storage volume is required for future EC2 workloads. For this task, an Amazon Elastic Block Store (EBS) volume was created with a specific name, volume type, and size using the AWS CLI.

| Requirement | Value |
| :--- | :--- |
| **Volume Name** | `nautilus-volume` |
| **Volume Type** | `gp3` |
| **Volume Size** | 2 GiB |
| **AWS Region** | `us-east-1` |
| **Method** | AWS CLI |

---

## 🎯 Objective

Create an EBS volume named `nautilus-volume` with:
* **Volume Type:** `gp3`
* **Size:** 2 GiB
* **Region:** `us-east-1`

The volume must be created within an active Availability Zone in `us-east-1`.

---

## 🛠️ Prerequisites

Make sure the AWS CLI is installed:
```bash
aws --version
```

Retrieve the temporary lab credentials:
```bash
showcreds
```

Verify the AWS identity:
```bash
aws sts get-caller-identity
```

Set and verify the required AWS region:
```bash
export AWS_DEFAULT_REGION=us-east-1
aws configure get region
```

**Expected output:**
```text
us-east-1
```

---

## 1. Find an Availability Zone

EBS volumes are bound to a specific Availability Zone (AZ). List the active Availability Zones:
```bash
aws ec2 describe-availability-zones \
  --filters "Name=state,Values=available" \
  --query "AvailabilityZones[].ZoneName" \
  --output text
```

**Example output:**
```text
us-east-1a us-east-1b us-east-1c
```

Select an Availability Zone:
```bash
AZ=us-east-1a
```

---

## 2. Create the EBS Volume

Create the required 2 GiB `gp3` volume with the specified tag:
```bash
aws ec2 create-volume \
  --volume-type gp3 \
  --size 2 \
  --availability-zone "$AZ" \
  --tag-specifications 'ResourceType=volume,Tags=[{Key=Name,Value=nautilus-volume}]'
```

**Example output:**
```json
{
    "AvailabilityZone": "us-east-1a",
    "CreateTime": "2026-01-15T12:00:00.000Z",
    "Encrypted": false,
    "Size": 2,
    "SnapshotId": "",
    "State": "creating",
    "VolumeId": "vol-0123456789abcdef0",
    "VolumeType": "gp3"
}
```

Save the returned volume ID:
```bash
VOLUME_ID=vol-0123456789abcdef0
```

---

## 3. Wait for the Volume to Become Available

Immediately after creation, the volume state may initially show as `creating`. Monitor until it transitions to `available`:
```bash
aws ec2 describe-volumes \
  --volume-ids "$VOLUME_ID" \
[O  --query "Volumes[].{VolumeId:VolumeId,State:State,Size:Size,Type:VolumeType,AZ:AvailabilityZone}" \
  --output table
```

---

## 4. Verify the Volume Configuration

Verify the volume by its Name tag:
```bash
aws ec2 describe-volumes \
  --filters "Name=tag:Name,Values=nautilus-volume" \
  --query "Volumes[].{Name:Tags[?Key=='Name']|[0].Value,VolumeId:VolumeId,Type:VolumeType,Size:Size,State:State,AZ:AvailabilityZone}" \
  --output table
```

Confirm individual properties using targeted query strings:

```bash
# Check Volume Type (Expected: gp3)
aws ec2 describe-volumes --volume-ids "$VOLUME_ID" --query "Volumes[0].VolumeType" --output text

# Check Volume Size (Expected: 2)
aws ec2 describe-volumes --volume-ids "$VOLUME_ID" --query "Volumes[0].Size" --output text

# Check Volume State (Expected: available)
aws ec2 describe-volumes --volume-ids "$VOLUME_ID" --query "Volumes[0].State" --output text
```

---

## 🔍 Verify Everything in One Command

Run a complete summary query prior to task submission:
```bash
aws ec2 describe-volumes \
  --filters "Name=tag:Name,Values=nautilus-volume" \
  --query "Volumes[].{Name:Tags[?Key=='Name']|[0].Value,ID:VolumeId,Type:VolumeType,SizeGiB:Size,AZ:AvailabilityZone,State:State}" \
  --output table
```

---

## ⚠️ Important: Availability Zones

EBS volumes are bound to a single Availability Zone:

```text
AWS Region (us-east-1)
 └── Availability Zone (us-east-1a)
      ├── EC2 Instance
      └── EBS Volume (nautilus-volume)
```

* An EBS volume can **only** be attached directly to an EC2 instance residing in the exact same Availability Zone (`us-east-1a`).
* To move storage to a different AZ (`us-east-1b`), create an EBS snapshot of the volume and restore it as a new volume in the target AZ.

---

## 💰 Why gp3?

`gp3` is a general-purpose SSD volume type offering key architectural benefits over older `gp2` volumes:
* **Decoupled Performance:** IOPS and throughput can be scaled independently of storage capacity without needing to provision extra storage space.
* **Cost Efficiency:** Delivers a lower baseline price per GB compared to `gp2`.

---

## 🔐 Security Considerations

EBS volumes frequently host sensitive database files or application states. For production environments, enable encryption at rest:

```bash
aws ec2 create-volume \
  --volume-type gp3 \
  --size 2 \
  --availability-zone "$AZ" \
  --encrypted \
  --tag-specifications 'ResourceType=volume,Tags=[{Key=Name,Value=nautilus-volume}]'
```

---

## 🧠 What I Learned & Key Takeaways

* AWS decouples compute (EC2) from persistent block storage (EBS), allowing disks to persist independently of instance lifecycles.
* Creating storage requires evaluating key parameters: **Volume Type → Capacity → Availability Zone → Performance Limits → Encryption/Cost**.
* The AWS CLI streamlines volume provisioning, making disk management scriptable and reproducible.

> *This was Task 5 of 50 AWS tasks in the AWS phase of my KodeKloud 100 Days of Cloud Challenge.*
