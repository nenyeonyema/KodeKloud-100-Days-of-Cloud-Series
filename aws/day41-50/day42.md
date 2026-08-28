# AWS Task 42 — DynamoDB To-Do Application: Creating and Managing Tasks

## Overview

This project documents the creation of a DynamoDB table to store and manage tasks for a simple To-Do application. The team needed a fast, scalable, and serverless NoSQL database solution to hold tasks identified by a unique task ID, each with a description and a status field. The table was created, populated with two tasks, and verified to confirm correct data insertion.

---

## Architecture

```
AWS DynamoDB
    │
    ▼
Table: devops-tasks
    │
    ├── Primary Key: taskId (String)
    │
    ├── Task 1
    │     taskId      → "1"
    │     description → "Learn DynamoDB"
    │     status      → "completed"
    │
    └── Task 2
          taskId      → "2"
          description → "Build To-Do App"
          status      → "in-progress"
```

---

## Prerequisites

- AWS CLI configured with appropriate credentials
- IAM permissions for DynamoDB operations:
  - `dynamodb:CreateTable`
  - `dynamodb:PutItem`
  - `dynamodb:GetItem`
  - `dynamodb:Scan`
- Region: `us-east-1`

---

## What Was Built

A DynamoDB table named `devops-tasks` with:
- A **partition key** of `taskId` (String type)
- **PAY_PER_REQUEST** billing mode — no capacity planning required
- Two task items inserted with description and status attributes
- Verification of both items confirming correct status values

---

## DynamoDB Item Structure

```
┌─────────────────────────────────────────────────┐
│              devops-tasks (Table)                │
│                                                  │
│  taskId (PK)  │  description      │  status      │
│  ─────────────────────────────────────────────   │
│  "1"          │  "Learn DynamoDB" │  "completed" │
│  "2"          │  "Build To-Do App"│  "in-progress│
└─────────────────────────────────────────────────┘
```

---

## Step-by-Step Implementation

### 1. Verify AWS Credentials

```bash
aws sts get-caller-identity
```

### 2. Create the DynamoDB Table

```bash
aws dynamodb create-table \
  --table-name devops-tasks \
  --attribute-definitions AttributeName=taskId,AttributeType=S \
  --key-schema AttributeName=taskId,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region us-east-1
```

### 3. Wait for Table to Become Active

```bash
aws dynamodb wait table-exists \
  --table-name devops-tasks \
  --region us-east-1

echo "✅ Table is now ACTIVE"
```

### 4. Insert Task 1

```bash
aws dynamodb put-item \
  --table-name devops-tasks \
  --item '{
    "taskId": {"S": "1"},
    "description": {"S": "Learn DynamoDB"},
    "status": {"S": "completed"}
  }' \
  --region us-east-1
```

### 5. Insert Task 2

```bash
aws dynamodb put-item \
  --table-name devops-tasks \
  --item '{
    "taskId": {"S": "2"},
    "description": {"S": "Build To-Do App"},
    "status": {"S": "in-progress"}
  }' \
  --region us-east-1
```

### 6. Verify Task 1

```bash
aws dynamodb get-item \
  --table-name devops-tasks \
  --key '{"taskId": {"S": "1"}}' \
  --region us-east-1
```

### 7. Verify Task 2

```bash
aws dynamodb get-item \
  --table-name devops-tasks \
  --key '{"taskId": {"S": "2"}}' \
  --region us-east-1
```

### 8. Full Table Scan to Confirm All Items

```bash
aws dynamodb scan \
  --table-name devops-tasks \
  --region us-east-1
```

---

## Verification

**Expected output for Task 1:**
```json
{
    "Item": {
        "taskId": { "S": "1" },
        "description": { "S": "Learn DynamoDB" },
        "status": { "S": "completed" }
    }
}
```

**Expected output for Task 2:**
```json
{
    "Item": {
        "taskId": { "S": "2" },
        "description": { "S": "Build To-Do App" },
        "status": { "S": "in-progress" }
    }
}
```

Both items confirmed ✅

---

## Console Verification Steps

For those who prefer the AWS Console:

1. **DynamoDB → Tables** — confirm `devops-tasks` table exists with status `Active`
2. **DynamoDB → Tables → devops-tasks → Explore items** — confirm both items are present
3. **Check taskId `1`** — status should be `completed`
4. **Check taskId `2`** — status should be `in-progress`

---

## Troubleshooting Checklist

| Check | Command | Expected Result |
|---|---|---|
| Table exists | `describe-table` | Status: `ACTIVE` |
| Task 1 inserted | `get-item` with key `"1"` | Returns item with status `completed` |
| Task 2 inserted | `get-item` with key `"2"` | Returns item with status `in-progress` |
| All items present | `scan` | Returns 2 items total |
| Billing mode | `describe-table` | `PAY_PER_REQUEST` |

---

## Common Mistakes

| Mistake | Consequence |
|---|---|
| Using `file://` instead of correct JSON inline | CLI rejects malformed input |
| Not waiting for table `ACTIVE` state before inserting | `PutItem` fails with `ResourceNotFoundException` |
| Wrong attribute type — using `N` instead of `S` for taskId | Key schema mismatch error on insert |
| Using `--provisioned-throughput` with `PAY_PER_REQUEST` | CLI throws validation error — not required in on-demand mode |
| Running `scan` on large tables in production | High read cost — use `query` with key conditions instead |

---

## Key Lesson

DynamoDB is schema-less for non-key attributes — only the primary key (`taskId`) needs to be declared at table creation time. All other attributes (`description`, `status`) are defined at write time per item, giving you full flexibility to evolve your data model without migrations.

Always follow this order when setting up DynamoDB:

```
Create Table → Wait for ACTIVE → Insert Items → Verify with get-item → Scan to confirm all
```

Never insert items before the table reaches `ACTIVE` state — the operation will fail silently or throw a resource error.

---

## Technologies Used

- **AWS DynamoDB** — Fully managed serverless NoSQL database
- **AWS CLI** — Table creation, item insertion and verification
- **PAY_PER_REQUEST Billing** — On-demand capacity with no provisioning
- **Partition Key Design** — `taskId` as the unique identifier per task
