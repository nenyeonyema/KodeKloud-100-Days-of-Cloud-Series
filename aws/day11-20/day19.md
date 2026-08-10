# AWS Task 19 — Attach IAM Policy to IAM User

## 📌 Task Overview

In this task, the Nautilus DevOps team required attaching an existing customer-managed IAM policy (`iampolicy_mark`) directly to an existing IAM user (`iamuser_mark`). The objective was to bind the read-only EC2 permissions defined in Task 18 to the user identity using the AWS CLI and verify the active policy attachment.

| Requirement | Value |
| :--- | :--- |
| **IAM User** | `iamuser_mark` |
| **IAM Policy Name** | `iampolicy_mark` |
| **AWS Region Context** | `us-east-1` (Note: IAM is a global service) |
| **Attachment Type** | Direct User Managed Policy Attachment |
| **Method** | AWS CLI |

---

## 🎯 Objective

Bind a managed policy permission boundary to a target user identity:

```text
Verify User & Policy Existence ──► Dynamically Resolve Policy ARN ──► aws iam attach-user-policy ──► Verify Attached User Policies
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

## 1. Verify User & Policy Existence

Confirm that target user `iamuser_mark` exists:

```bash
aws iam get-user \
  --user-name iamuser_mark \
  --query "User.{UserName:UserName,UserId:UserId,Arn:Arn}" \
  --output table
```

Dynamically resolve and store the ARN for `iampolicy_mark`:

```bash
POLICY_ARN=$(aws iam list-policies \
  --scope Local \
  --query "Policies[?PolicyName=='iampolicy_mark'].Arn" \
  --output text)

echo "Resolved Policy ARN: $POLICY_ARN"
```

**Expected output:**
```text
Resolved Policy ARN: arn:aws:iam::123456789012:policy/iampolicy_mark
```

---

## 2. Attach the IAM Policy to the User

Attach `iampolicy_mark` directly to `iamuser_mark`:

```bash
aws iam attach-user-policy \
  --user-name iamuser_mark \
  --policy-arn "$POLICY_ARN"
```

*(Note: The AWS CLI produces no stdout payload upon successful attachment execution.)*

---

## 3. Verify Policy Attachment

Query the managed policies directly attached to `iamuser_mark`:

```bash
aws iam list-attached-user-policies \
  --user-name iamuser_mark \
  --query "AttachedPolicies[?PolicyName=='iampolicy_mark'].{PolicyName:PolicyName,PolicyArn:PolicyArn}" \
  --output table
```

**Expected output:**
```text
--------------------------------------------------------------------------------------------------
|                                    ListAttachedUserPolicies                                    |
+----------------------------------------------------+-------------------------------------------+
|                     PolicyArn                      |                PolicyName                 |
+----------------------------------------------------+-------------------------------------------+
|  arn:aws:iam::123456789012:policy/iampolicy_mark  |  iampolicy_mark                           |
+----------------------------------------------------+-------------------------------------------+
```

---

## 🔍 Complete Verification

Perform a quick check returning the single policy name:

```bash
aws iam list-attached-user-policies \
  --user-name iamuser_mark \
  --query "AttachedPolicies[?PolicyName=='iampolicy_mark'].PolicyName" \
  --output text
```

**Expected output:**
```text
iampolicy_mark
```

---

## 🧠 Key Concepts & Takeaways

| Concept | Description |
| :--- | :--- |
| **Explicit Attachment Requirement** | Creating an IAM policy defines permissions but grants no actual access until explicitly bound to an identity (User, Group, or Role). |
| **Direct vs. Group Attachment** | Direct user attachment (`attach-user-policy`) works well for singular edge cases. Group attachment (`attach-group-policy`) is preferred for scaling access across teams. |
| **Idempotency** | The `attach-user-policy` command is idempotent; running it multiple times against the same user and policy ARN succeeds without creating duplicate attachments. |
| **Effective Access** | Once attached, `iamuser_mark` immediately inherits the permissions defined in `iampolicy_mark` (e.g., `ec2:DescribeInstances`, `ec2:DescribeImages`, `ec2:DescribeSnapshots`). |

> *This was Task 19 of 50 AWS tasks in the AWS phase of my KodeKloud 100 Days of Cloud Challenge.*
