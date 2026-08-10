# AWS Task 16 — Create an IAM User

## 📌 Task Overview

In this task, the Nautilus DevOps team required a new Identity and Access Management (IAM) user to support identity management requirements. The objective was to create a global IAM user named `iamuser_rose` using the AWS CLI and verify its creation and resource metadata.

| Requirement | Value |
| :--- | :--- |
| **IAM User Name** | `iamuser_rose` |
| **AWS Region Context** | `us-east-1` (Note: IAM is a global service) |
| **Target User State** | Created and verified |
| **Method** | AWS CLI |

---

## 🎯 Objective

Provision a long-term IAM identity entity at the global account level:

```text
Check Existence (NoSuchEntity) ──► aws iam create-user ──► Retrieve User ARN & Unique ID ──► Verify in Account Listing
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

## 1. Verify Non-Existence of the IAM User

Check whether the target user entity already exists:

```bash
aws iam get-user \
  --user-name iamuser_rose
```

**Expected output (if user does not exist):**
```text
An error occurred (NoSuchEntity) when calling the GetUser operation: The user with name iamuser_rose cannot be found.
```

---

## 2. Create the IAM User

Issue the user creation API call:

```bash
aws iam create-user \
  --user-name iamuser_rose
```

**Example JSON response:**
```json
{
    "User": {
        "Path": "/",
        "UserName": "iamuser_rose",
        "UserId": "AIDAEXAMPLE123456",
        "Arn": "arn:aws:iam::123456789012:user/iamuser_rose",
        "CreateDate": "2026-08-10T20:24:16Z"
    }
}
```

---

## 3. Retrieve and Verify IAM User Attributes

Query the newly created user's essential metadata:

```bash
aws iam get-user \
  --user-name iamuser_rose \
  --query "User.{UserName:UserName,UserId:UserId,Arn:Arn}" \
  --output table
```

**Expected output:**
```text
---------------------------------------------------------------------------------------
|                                       GetUser                                       |
+-------------------+-----------------------------------+-----------------------------+
|        Arn        |              UserId               |          UserName           |
+-------------------+-----------------------------------+-----------------------------+
|  arn:aws:iam::... |  AIDAEXAMPLE123456                |  iamuser_rose               |
+-------------------+-----------------------------------+-----------------------------+
```

---

## 4. Account-Wide IAM User Listing

Confirm that `iamuser_rose` is present in the account-level IAM user directory:

```bash
aws iam list-users \
  --query "Users[?UserName=='iamuser_rose'].UserName" \
  --output text
```

**Expected output:**
```text
iamuser_rose
```

---

## 🔍 Complete Verification

Perform an all-in-one explicit existence check:

```bash
aws iam get-user \
  --user-name iamuser_rose \
  --query "User.UserName" \
  --output text
```

**Expected output:**
```text
iamuser_rose
```

---

## 🧠 Key Concepts & Takeaways

| Concept | Description |
| :--- | :--- |
| **Global Infrastructure** | Unlike regional resources (e.g., EC2, EBS, VPC), IAM operates globally at the account level. Commands affect the root IAM endpoint across all regions. |
| **Authentication vs. Authorization** | `create-user` provisions an identity (authentication capability) but grants zero default access (authorization). Permissions must be attached separately via IAM Policies, Groups, or Roles. |
| **Principle of Least Privilege** | Identities should start with an implicit deny across all actions and be granted only the explicit, scope-limited permissions required for their specific function. |
| **Unique Identifiers** | Each IAM user receives an Amazon Resource Name (`Arn`) and a unique 21-character alphanumeric identifier starting with `AIDA` (`UserId`). |

> *This was Task 16 of 50 AWS tasks in the AWS phase of my KodeKloud 100 Days of Cloud Challenge.*
