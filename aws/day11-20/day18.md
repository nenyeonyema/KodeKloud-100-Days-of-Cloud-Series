# AWS Task 18 — Create an IAM Policy

## 📌 Task Overview

In this task, the Nautilus DevOps team required a custom IAM managed policy to enforce read-only access for core EC2 components. The objective was to create a customer-managed policy named `iampolicy_mark` that grants permission to view EC2 instances, AMIs, and EBS snapshots without permitting any state-modifying or destructive operations.

| Requirement | Value |
| :--- | :--- |
| **IAM Policy Name** | `iampolicy_mark` |
| **Access Level** | Read-Only (`Describe` actions) |
| **Target Service Scope** | EC2 Instances, AMIs, EBS Snapshots |
| **AWS Region Context** | `us-east-1` (Note: IAM is a global service) |
| **Method** | AWS CLI & JSON Policy Document |

---

## 🎯 Objective

Provision a reusable customer-managed IAM policy defining scope-limited read permissions:

```text
Define JSON Policy Document ──► aws iam create-policy ──► Inspect Policy Version ──► Verify Local Scope Listing
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

## 1. Create the Policy Document

Define the policy document defining explicit read permissions for EC2 instances, AMIs, and snapshots:

```bash
cat << 'EOF' > ec2-readonly-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeInstances",
        "ec2:DescribeImages",
        "ec2:DescribeSnapshots"
      ],
      "Resource": "*"
    }
  ]
}
EOF
```

---

## 2. Create the Customer-Managed IAM Policy

Execute the policy creation command pointing to the local JSON document:

```bash
aws iam create-policy \
  --policy-name iampolicy_mark \
  --policy-document file://ec2-readonly-policy.json
```

**Example JSON response:**
```json
{
    "Policy": {
        "PolicyName": "iampolicy_mark",
        "PolicyId": "ANPAEXAMPLE123456",
        "Arn": "arn:aws:iam::123456789012:policy/iampolicy_mark",
        "Path": "/",
        "DefaultVersionId": "v1",
        "AttachmentCount": 0,
        "IsAttachable": true,
        "CreateDate": "2026-08-10T20:28:12Z",
        "UpdateDate": "2026-08-10T20:28:12Z"
    }
}
```

Save the generated Policy ARN for verification:
```bash
POLICY_ARN=$(aws iam list-policies --scope Local --query "Policies[?PolicyName=='iampolicy_mark'].Arn" --output text)
```

---

## 3. Retrieve and Verify Policy Metadata

Query the basic metadata of the newly registered policy:

```bash
aws iam get-policy \
  --policy-arn "$POLICY_ARN" \
  --query "Policy.{Name:PolicyName,Arn:Arn,Id:PolicyId,DefaultVersion:DefaultVersionId}" \
  --output table
```

**Expected output:**
```text
------------------------------------------------------------------------------------------------
|                                           GetPolicy                                          |
+-----------------+----------------------------------------------------+-----------------------+
| DefaultVersion  |                        Arn                         |          Id           |
+-----------------+----------------------------------------------------+-----------------------+
|  v1             |  arn:aws:iam::123456789012:policy/iampolicy_mark  |  ANPAEXAMPLE123456    |
+-----------------+----------------------------------------------------+-----------------------+
```

---

## 4. Inspect Policy Document Statements

Verify that default version `v1` enforces the required read-only permission scope:

```bash
aws iam get-policy-version \
  --policy-arn "$POLICY_ARN" \
  --version-id v1 \
  --query "PolicyVersion.Document.Statement[].Action" \
  --output json
```

**Expected output:**
```json
[
    "ec2:DescribeInstances",
    "ec2:DescribeImages",
    "ec2:DescribeSnapshots"
]
```

---

## 🔍 Complete Verification

Verify that the policy appears in the customer-managed policy catalog (`--scope Local`):

```bash
aws iam list-policies \
  --scope Local \
  --query "Policies[?PolicyName=='iampolicy_mark'].{Name:PolicyName,Arn:Arn}" \
  --output table
```

---

## 🧠 Key Concepts & Takeaways

| Concept | Description |
| :--- | :--- |
| **Policy Grammar** | Structured JSON defining permissions using `Effect` (`Allow`/`Deny`), `Action` (API calls), `Resource` (ARNs), and optional `Condition` blocks. |
| **Describe Actions** | EC2 read operations follow the `ec2:Describe*` naming pattern. These APIs do not support resource-level permissions, requiring `"Resource": "*"`. |
| **Customer Managed Policies** | Reusable policies created within an account (`--scope Local`) that offer fine-grained access control compared to broad AWS Managed Policies. |
| **Policy Versioning** | IAM tracks changes to policies via versions (`v1`, `v2`, etc.), allowing updates or rollbacks without detaching the policy from identities. |
| **Decoupled Security** | Creating a policy defines a permission boundary, but grants no access until attached to an IAM User, Group, or Role (`attach-user-policy`, `attach-group-policy`). |

> *This was Task 18 of 50 AWS tasks in the AWS phase of my KodeKloud 100 Days of Cloud Challenge.*
