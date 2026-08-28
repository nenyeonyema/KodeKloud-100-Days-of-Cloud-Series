# AWS Task 31 — Create a Private RDS MySQL Instance

## 📌 Task Overview

The Nautilus Development Team required an isolated database instance for application development and testing. The objective was to provision a private MySQL 8.4.x Amazon RDS instance (`devops-rds`) using the cost-effective `db.t3.micro` instance class. The database was placed inside a dedicated private DB subnet group with public accessibility explicitly disabled (`--no-publicly-accessible`) and storage autoscaling enabled up to a maximum threshold of 50 GB (`gp3`).

| Requirement | Value |
| :--- | :--- |
| **DB Instance Identifier** | `devops-rds` |
| **Engine / Version** | `mysql` (8.4.x) |
| **DB Instance Class** | `db.t3.micro` |
| **Storage Config** | 20 GB initial (`gp3`), Autoscaling Max: `50` GB |
| **Public Accessibility** | Private (`false`) |
| **DB Subnet Group** | `devops-db-subnet-group` |
| **AWS Region Context** | `us-east-1` |
| **Target State** | `available` |

---

## 🎯 Architecture & Access Boundary

```text
                      AWS Region: us-east-1
                                 │
                                 ▼
                     Private VPC / Subnets
                                 │
                     ┌───────────────────────┐
                     │  DB Subnet Group      │
                     │ devops-db-subnet-group│
                     └───────────┬───────────┘
                                 │
                                 ▼
                     ┌───────────────────────┐
                     │      Amazon RDS       │
                     │      devops-rds       │
                     │  (Publicly Acc: false)│
                     │                       │
                     │     MySQL 8.4.x       │
                     │     db.t3.micro       │
                     └───────────┬───────────┘
                                 │
                                 ▼
                        Storage Autoscaling
                       (20 GB ──► Max 50 GB)
🛠️ Implementation & Verification Commands
Bash
# ==========================================
# 1. CONFIGURE REGION & DISCOVER DB SUBNET GROUP
# ==========================================
aws configure set region us-east-1

# Identify Available Private DB Subnet Group
DB_SUBNET_GROUP=$(aws rds describe-db-subnet-groups \
  --region us-east-1 \
  --query "DBSubnetGroups[0].DBSubnetGroupName" \
  --output text)

echo "Target DB Subnet Group: ${DB_SUBNET_GROUP}"

# Retrieve Supported MySQL 8.4.x Engine Version
ENGINE_VERSION=$(aws rds describe-db-engine-versions \
  --engine mysql \
  --region us-east-1 \
  --query "DBEngineVersions[?starts_with(EngineVersion, '8.4')].EngineVersion | [0]" \
  --output text)

echo "Target MySQL Engine Version: ${ENGINE_VERSION}"

# ==========================================
# 2. PROVISION PRIVATE RDS MYSQL INSTANCE
# ==========================================
aws rds create-db-instance \
  --db-instance-identifier devops-rds \
  --db-instance-class db.t3.micro \
  --engine mysql \
  --engine-version "$ENGINE_VERSION" \
  --allocated-storage 20 \
  --max-allocated-storage 50 \
  --storage-type gp3 \
  --db-subnet-group-name "$DB_SUBNET_GROUP" \
  --no-publicly-accessible \
  --master-username admin \
  --manage-master-user-password \
  --backup-retention-period 0 \
  --region us-east-1

# ==========================================
# 3. WAIT FOR RDS INSTANCE PROVISIONING
# ==========================================
echo "Waiting for RDS instance devops-rds to enter 'available' state..."

aws rds wait db-instance-available \
  --db-instance-identifier devops-rds \
  --region us-east-1

# ==========================================
# 4. VERIFICATION & INTEGRITY CHECKS
# ==========================================
# Verify Private Status, Class, Engine, and Autoscaling Storage Threshold
aws rds describe-db-instances \
  --db-instance-identifier devops-rds \
  --region us-east-1 \
  --query "DBInstances[0].{Identifier:DBInstanceIdentifier, Status:DBInstanceStatus, Class:DBInstanceClass, Engine:Engine, Version:EngineVersion, PubliclyAccessible:PubliclyAccessible, AllocatedStorage:AllocatedStorage, MaxStorage:MaxAllocatedStorage}" \
  --output table
Note: A returned status of available alongside PubliclyAccessible: false and MaxStorage: 50 confirms that devops-rds is properly provisioned inside the private subnet group with automated storage capacity expansion configured.
