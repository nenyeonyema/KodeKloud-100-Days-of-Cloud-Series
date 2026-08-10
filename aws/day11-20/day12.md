# AWS Task 12 — Attach an EBS Volume to an EC2 Instance

## 📌 Task Overview

In this task, the Nautilus DevOps team required an existing Elastic Block Store (EBS) volume in `us-east-1` to be attached to an existing EC2 instance as block storage using a specific block device mapping (`/dev/sdb`). The task was executed and validated entirely using the AWS CLI.

| Requirement | Value |
| :--- | :--- |
| **EC2 Instance** | `datacenter-ec2` |
| **EBS Volume** | `datacenter-volume` |
| **Device Name** | `/dev/sdb` |
| **AWS Region** | `us-east-1` |
| **Target Volume State** | `in-use` |
| **Method** | AWS CLI |

---

## 🎯 Objective

Attach the unattached EBS volume to the targeted EC2 instance with the requested device mapping:

```text
EBS Volume (datacenter-volume) ──► Attach (/dev/sdb) ──► EC2 Instance (datacenter-ec2)
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
  --filters "Name=tag:Name,Values=datacenter-ec2" \
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

> **Note:** If the instance is stopped, start it and wait for the `running` state before attaching storage:
> ```bash
> aws ec2 start-instances --instance-ids "$INSTANCE_ID"
> aws ec2 wait instance-running --instance-ids "$INSTANCE_ID"
> ```

---

## 2. Retrieve and Inspect the EBS Volume

Locate the Volume ID using the `Name` tag filter:
```bash
aws ec2 describe-volumes \
  --filters "Name=tag:Name,Values=datacenter-volume" \
  --query "Volumes[].VolumeId" \
  --output text
```

**Example output:**
```text
vol-0123456789abcdef0
```

Save the Volume ID:
```bash
VOLUME_ID=vol-0123456789abcdef0
```

Verify that the volume state is `available` (unattached):
```bash
aws ec2 describe-volumes \
  --volume-ids "$VOLUME_ID" \
  --query "Volumes[].{VolumeId:VolumeId,State:State,Size:Size,Type:VolumeType}" \
  --output table
```

---

## 3. Attach the EBS Volume

Attach the volume to the target EC2 instance mapped to `/dev/sdb`:

```bash
aws ec2 attach-volume \
  --volume-id "$VOLUME_ID" \
  --instance-id "$INSTANCE_ID" \
  --device /dev/sdb
```

**Example JSON response:**
```json
{
    "AttachTime": "2026-08-10T20:09:30.000Z",
    "Device": "/dev/sdb",
    "InstanceId": "i-0123456789abcdef0",
    "State": "attaching",
    "VolumeId": "vol-0123456789abcdef0"
}
```

Wait until the volume state transitions from `attaching` to `in-use`:
```bash
aws ec2 wait volume-in-use \
  --volume-ids "$VOLUME_ID"
```

---

## 4. Verify Volume Attachment

Confirm that the volume state shows `in-use` and reflects the correct device mapping and instance association:

```bash
aws ec2 describe-volumes \
  --volume-ids "$VOLUME_ID" \
  --query "Volumes[].{VolumeId:VolumeId,State:State,InstanceId:Attachments[0].InstanceId,Device:Attachments[0].Device}" \
  --output table
```

**Expected output:**
```text
---------------------------------------------------------
|                   DescribeVolumes                     |
+----------------+----------------------+---------------+
| Device         | /dev/sdb                             |
| InstanceId     | i-0123456789abcdef0                  |
| State          | in-use                               |
| VolumeId       | vol-0123456789abcdef0                |
+----------------+----------------------+---------------+
```

---

## 🔍 Complete Verification

Execute a final single-command query using tag filtering to validate the entire setup:

```bash
aws ec2 describe-volumes \
  --filters "Name=tag:Name,Values=datacenter-volume" \
  --query "Volumes[].{Volume:VolumeId,State:State,Instance:Attachments[0].InstanceId,Device:Attachments[0].Device}" \
  --output table
```

---

## 🧠 Key Concepts & Takeaways

| Concept | Description |
| :--- | :--- |
| **EBS Lifecycle** | EBS volumes are independent block storage resources that can be dynamically attached, detached, and reattached. |
| **Volume States** | `available` indicates an unattached volume ready for binding; `in-use` indicates an attached volume. |
| **Device Mapping** | Linux block device names like `/dev/sdb` map to kernel block devices (e.g., `/dev/xvdb` or `/dev/nvme1n1` depending on instance generation). |
| **Cost Management** | Unattached (`available`) EBS volumes continue incurring storage costs based on provisioned size; idle volumes should be snapshotted and removed. |

> *This was Task 12 of 50 AWS tasks in the AWS phase of my KodeKloud 100 Days of Cloud Challenge.*
