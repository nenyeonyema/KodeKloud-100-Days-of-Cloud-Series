# AWS Task 17 — Create an IAM Group

## 📌 Task Overview

In this task, the Nautilus DevOps team required a new Identity and Access Management (IAM) group to support scalable access control and permission management. The objective was to create a global IAM group named `iamgroup_ravi` using the AWS CLI and verify its creation and resource metadata.

| Requirement | Value |
| :--- | :--- |
| **IAM Group Name** | `iamgroup_ravi` |
| **AWS Region Context** | `us-east-1` (Note: IAM is a global service) |
| **Target Group State** | Created and verified |
| **Method** | AWS CLI |

---

## 🎯 Objective

Provision a long-term IAM group entity at the global account level to aggregate user permissions:

```text
Check Existence (NoSuchEntity) ──► aws iam create-group ──► Retrieve Group ARN & Unique ID ──► Verify in Group Directory
```

---

## 🛠️ Prerequisites

Set the active default region for the AWS CLI session:
```bash
export AWS_DEFAULT_REGION=us-east-1
aws configure get region
```

**Expected output:**
```text
us-east-1
```

---

## 1. Verify Non-Existence of the IAM Group

Check whether the target group entity already exists:

```bash
aws iam get-group \
  --group-name iamgroup_ravi
```

**Expected output (if group does not exist):**
```text
An error occurred (NoSuchEntity) when calling the GetGroup operation: The group with name iamgroup_ravi cannot be found.
```

---

## 2. Create the IAM Group

Issue the group creation API call:

```bash
aws iam create-group \
  --group-name iamgroup_ravi
```

**Example JSON response:**
```json
{
    "Group": {
        "Path": "/",
        "GroupName": "iamgroup_ravi",
        "GroupId": "AGPAEXAMPLE123456",
        "Arn": "arn:aws:iam::123456789012:group/iamgroup_ravi",
        "CreateDate": "2026-08-10T20:25:47Z"
    }
}
```

---

## 3. Retrieve and Verify IAM Group Attributes

Query the newly created group's essential metadata:

```bash
aws iam get-group \
  --group-name iamgroup_ravi \
  --query "Group.{GroupName:GroupName,GroupId:GroupId,Arn:Arn}" \
  --output table
```

**Expected output:**
```text
---------------------------------------------------------------------------------------
|                                       GetGroup                                      |
+-------------------+-----------------------------------+-----------------------------+
|        Arn        |              GroupId              |          GroupName          |
+-------------------+-----------------------------------+-----------------------------+
|  arn:aws:iam::... |  AGPAEXAMPLE123456                |  iamgroup_ravi              |
+-------------------+-----------------------------------+-----------------------------+
```

---

## 4. Account-Wide IAM Group Listing

Confirm that `iamgroup_ravi` appears in the account-level IAM group directory:

```bash
aws iam list-groups \
  --query "Groups[?GroupName=='iamgroup_ravi'].GroupName" \
  --output text
```

**Expected output:**
```text
iamgroup_ravi
```

---

## 🔍 Complete Verification

Perform an all-in-one explicit existence check:

```bash
aws iam get-group \
  --group-name iamgroup_ravi \
  --query "Group.GroupName" \
  --output text
```

**Expected output:**
```text
iamgroup_ravi
```

---

## 🧠 Key Concepts & Takeaways

| Concept | Description |
| :--- | :--- |
| **Group-Based Access Control** | IAM groups organize multiple IAM users into manageable entities. Attaching permissions to groups rather than individual users prevents permission drift and simplifies auditing. |
| **Global Scope** | Like IAM Users, Roles, and Policies, IAM Groups exist at the global account level and operate across all AWS regions simultaneously. |
| **Unique Identifiers** | Each IAM group receives an Amazon Resource Name (`Arn`) and a unique 21-character alphanumeric identifier starting with `AGPA` (`GroupId`). |
| **Core IAM Primitives** | **User** (Who is accessing), **Group** (Who shares job functions), **Policy** (What permissions are granted), **Role** (Temporary identity assumed by users/services). |

> *This was Task 17 of 50 AWS tasks in the AWS phase of my KodeKloud 100 Days of Cloud Challenge.*
