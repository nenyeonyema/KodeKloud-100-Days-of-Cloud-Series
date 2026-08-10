# AWS Task 15 — Create an EBS Snapshot

## 📌 Task Overview

In this task, the Nautilus DevOps team required an automated backup solution for critical block storage. The objective was to create a point-in-time Elastic Block Store (EBS) snapshot named `datacenter-vol-ss` from an existing volume (`datacenter-vol`) in `us-east-1` with a specific description, apply proper tagging, and verify that the snapshot reached the `completed` state using the AWS CLI.

| Requirement | Value |
| :--- | :--- |
| **EBS Volume** | `datacenter-vol` |
| **Snapshot Name Tag** | `datacenter-vol-ss` |
| **Snapshot Description** | `datacenter Snapshot` |
| **AWS Region** | `us-east-1` |
| **Target Snapshot State** | `completed` |
| **Method** | AWS CLI |

---

## 🎯 Objective

Generate a point-in-time point-of-recovery block storage backup:

```text
EBS Volume (datacenter-vol) ──► create-snapshot ──► Apply Name Tag ──► wait snapshot-completed ──► State: completed
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

## 1. Retrieve and Inspect the EBS Volume

Locate the target Volume ID using the `Name` tag filter:
```bash
aws ec2 describe-volumes \
  --filters "Name=tag:Name,Values=datacenter-vol" \
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

Inspect the volume state (`in-use` or `available`):
```bash
aws ec2 describe-volumes \
  --volume-ids "$VOLUME_ID" \
  --query "Volumes[].{VolumeId:VolumeId,State:State,Size:Size,Type:VolumeType}" \
  --output table
```

---

## 2. Create the EBS Snapshot

Issue the snapshot creation command with the required description string:

```bash
aws ec2 create-snapshot \
  --volume-id "$VOLUME_ID" \
  --description "datacenter Snapshot"
```

**Example JSON response:**
```json
{
    "Description": "datacenter Snapshot",
    "SnapshotId": "snap-0123456789abcdef0",
    "State": "pending",
    "VolumeId": "vol-0123456789abcdef0"
}
```

Save the returned Snapshot ID:
```bash
SNAPSHOT_ID=snap-0123456789abcdef0
```

---

## 3. Apply the Snapshot Name Tag

Apply the explicit `Name` tag resource key to fulfill the naming requirement:

```bash
aws ec2 create-tags \
  --resources "$SNAPSHOT_ID" \
  --tags Key=Name,Value=datacenter-vol-ss
```

---

## 4. Monitor and Wait for Snapshot Completion

Check the initial state of the snapshot (`pending`):

```bash
aws ec2 describe-snapshots \
  --snapshot-ids "$SNAPSHOT_ID" \
  --query "Snapshots[].{SnapshotId:SnapshotId,State:State,Description:Description}" \
  --output table
```

Block execution until AWS finishes writing block deltas and the state transitions to `completed`:

```bash
aws ec2 wait snapshot-completed \
  --snapshot-ids "$SNAPSHOT_ID"
```

---

## 🔍 Complete Verification

Run an all-in-one tag query to validate the name, description, and state:

```bash
aws ec2 describe-snapshots \
  --filters "Name=tag:Name,Values=datacenter-vol-ss" \
  --query "Snapshots[].{SnapshotId:SnapshotId,Name:Tags[?Key=='Name']|[0].Value,State:State,Description:Description,VolumeId:VolumeId}" \
  --output table
```

**Expected output:**
```text
-----------------------------------------------------------------------------------------
|                                  DescribeSnapshots                                    |
+----------------------+----------------------+---------------------+-------------------+
|  Description         |  Name                |  SnapshotId         |  State            |
+----------------------+----------------------+---------------------+-------------------+
|  datacenter Snapshot |  datacenter-vol-ss   |  snap-0123456789... |  completed        |
+----------------------+----------------------+---------------------+-------------------+
```

---

## 🧠 Key Concepts & Takeaways

| Concept | Description |
| :--- | :--- |
| **Point-in-Time Delta** | EBS snapshots are incremental backups (only blocks changed since the last snapshot are saved), optimizing both storage footprint and creation speed. |
| **Resource Tagging** | EBS Snapshots do not have a native display name parameter at creation; the resource name is applied post-creation via the `Name` tag (`Key=Name,Value=...`). |
| **Asynchronous Creation** | `create-snapshot` immediately returns a `pending` status while AWS streams volume data; script execution must use `aws ec2 wait snapshot-completed` to confirm readiness. |
| **Disaster Recovery (DR)** | Snapshots can be used to re-instantiate new EBS volumes across Availability Zones or copied across regions for cross-region disaster recovery. |

> *This was Task 15 of 50 AWS tasks in the AWS phase of my KodeKloud 100 Days of Cloud Challenge.*
