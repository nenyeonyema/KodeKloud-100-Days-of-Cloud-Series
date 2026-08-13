# AWS Task 32 — RDS Snapshot and Restoration

## 📌 Task Overview

The Nautilus Development Team required a backup and restoration workflow for database validation and testing. The objective was to capture a point-in-time backup of an existing operational Amazon RDS instance (`xfusion-rds`), store it as a manual database snapshot (`xfusion-snapshot`), and restore that snapshot into a brand new, independent target instance named `xfusion-snapshot-restore` using the `db.t3.micro` instance class in `us-east-1`.

| Requirement | Value |
| :--- | :--- |
| **Source RDS Instance** | `xfusion-rds` (State: `available`) |
| **DBSnapshot Identifier** | `xfusion-snapshot` |
| **Target Restored Instance** | `xfusion-snapshot-restore` |
| **Target DB Instance Class** | `db.t3.micro` |
| **AWS Region Context** | `us-east-1` |
| **Core Workflow** | Verify Source ──► Create Snapshot ──► Restore Instance ──► Validate Availability |

---

## 🎯 Architecture & Disaster Recovery Workflow

```text
               Existing Source RDS
                   xfusion-rds
                        │
                        │ aws rds create-db-snapshot
                        ▼
             DBSnapshot Artifact
              xfusion-snapshot
                        │
                        │ aws rds restore-db-instance-from-db-snapshot
                        ▼
            New Restored RDS Instance
            xfusion-snapshot-restore
               (Class: db.t3.micro)
                        │
                        │ aws rds wait db-instance-available
                        ▼
               Status: available
🛠️ Implementation & Verification Commands
Bash
# ==========================================
# 1. CONFIGURE REGION & VERIFY SOURCE INSTANCE
# ==========================================
aws configure set region us-east-1

# Confirm xfusion-rds exists and is available
aws rds describe-db-instances \
  --db-instance-identifier xfusion-rds \
  --region us-east-1 \
  --query "DBInstances[0].{Identifier:DBInstanceIdentifier, Status:DBInstanceStatus, Engine:Engine, Class:DBInstanceClass}" \
  --output table

# Wait for source instance availability if pending
aws rds wait db-instance-available \
  --db-instance-identifier xfusion-rds \
  --region us-east-1

# ==========================================
# 2. CREATE & VERIFY RDS DATABASE SNAPSHOT
# ==========================================
aws rds create-db-snapshot \
  --db-instance-identifier xfusion-rds \
  --db-snapshot-identifier xfusion-snapshot \
  --region us-east-1

echo "Waiting for snapshot xfusion-snapshot to complete..."

aws rds wait db-snapshot-available \
  --db-snapshot-identifier xfusion-snapshot \
  --region us-east-1

# Verify Snapshot State
aws rds describe-db-snapshots \
  --db-snapshot-identifier xfusion-snapshot \
  --region us-east-1 \
  --query "DBSnapshots[0].{Snapshot:DBSnapshotIdentifier, Status:Status, Engine:Engine, Source:DBInstanceIdentifier}" \
  --output table

# ==========================================
# 3. RESTORE SNAPSHOT TO NEW DB INSTANCE
# ==========================================
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier xfusion-snapshot-restore \
  --db-snapshot-identifier xfusion-snapshot \
  --db-instance-class db.t3.micro \
  --region us-east-1

# ==========================================
# 4. WAIT FOR RESTORED INSTANCE & VERIFY
# ==========================================
echo "Waiting for restored instance xfusion-snapshot-restore to reach 'available' state..."

aws rds wait db-instance-available \
  --db-instance-identifier xfusion-snapshot-restore \
  --region us-east-1

# Final Verification
aws rds describe-db-instances \
  --db-instance-identifier xfusion-snapshot-restore \
  --region us-east-1 \
  --query "DBInstances[0].{Identifier:DBInstanceIdentifier, Status:DBInstanceStatus, Class:DBInstanceClass, Engine:Engine, Version:EngineVersion}" \
  --output table
Note: Returning a status of available for xfusion-snapshot-restore with instance class db.t3.micro validates that the point-in-time snapshot restoration completed successfully and the database is ready for application usage.
