# AWS Task 23 — Data Migration Between S3 Buckets Using AWS CLI

## 📌 Task Overview

The objective of this task was to perform a complete server-side data migration from an existing Amazon S3 bucket (`xfusion-s3-17033`) to a newly created, private destination S3 bucket (`xfusion-sync-5342`) using the AWS CLI in `us-east-1`.

| Requirement | Value |
| :--- | :--- |
| **Source S3 Bucket** | `xfusion-s3-17033` |
| **Destination S3 Bucket** | `xfusion-sync-5342` |
| **AWS Region Context** | `us-east-1` |
| **Access Controls** | Private by default (no public bucket policy) |
| **Migration Tooling** | `aws s3 sync` & `aws s3api` CLI commands |

---

## 🎯 Architecture

```text
┌─────────────────────────┐
│   xfusion-s3-17033      │
│      Source Bucket      │
└────────────┬────────────┘
             │
             │ aws s3 sync
             ▼
┌─────────────────────────┐
│   xfusion-sync-5342     │
│   Private Destination   │
└─────────────────────────┘
🛠️ Implementation & Verification Commands
Bash
# ==========================================
# 1. CONFIGURE REGION & INSPECT SOURCE
# ==========================================
aws configure set region us-east-1
aws configure get region

# List source bucket contents and capture baseline summary
aws s3 ls s3://xfusion-s3-17033/ --recursive
aws s3 ls s3://xfusion-s3-17033/ --recursive --summarize

# ==========================================
# 2. CREATE & VERIFY DESTINATION BUCKET
# ==========================================
aws s3api create-bucket \
  --bucket xfusion-sync-5342 \
  --region us-east-1

# Verify bucket exists and confirm privacy setup
aws s3 ls
aws s3api get-public-access-block --bucket xfusion-sync-5342

# ==========================================
# 3. MIGRATE DATA (SERVER-SIDE SYNC)
# ==========================================
aws s3 sync \
  s3://xfusion-s3-17033 \
  s3://xfusion-sync-5342

# ==========================================
# 4. VERIFY & COMPARE DATA PARITY
# ==========================================
# List destination contents
aws s3 ls s3://xfusion-sync-5342/ --recursive

# Compare total object counts and total byte sizes
aws s3 ls s3://xfusion-s3-17033/ --recursive --summarize
aws s3 ls s3://xfusion-sync-5342/ --recursive --summarize

# Perform final dry-run synchronization check (should produce no output)
aws s3 sync \
  s3://xfusion-s3-17033 \
  s3://xfusion-sync-5342 \
  --dryrun
Note: Matching Total Objects and Total Size between the source and destination summaries, along with a clean --dryrun output, confirms 100% data migration parity.
