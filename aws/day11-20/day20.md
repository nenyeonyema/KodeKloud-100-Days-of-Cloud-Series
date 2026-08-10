# AWS Task 20 — Create an IAM Role for EC2

## 📌 Task Overview

In this task, the Nautilus DevOps team required an IAM role designed to grant an Amazon EC2 instance temporary AWS permissions without embedding long-term access keys into application code. The objective was to create an IAM role named `iamrole_ammar` with an EC2 service trust policy, attach an existing managed permission policy (`iampolicy_ammar`), and verify the complete role configuration using the AWS CLI.

| Requirement | Value |
| :--- | :--- |
| **IAM Role Name** | `iamrole_ammar` |
| **Trusted Entity** | AWS Service (`ec2.amazonaws.com`) |
| **Trust Action** | `sts:AssumeRole` |
| **Managed Permission Policy** | `iampolicy_ammar` |
| **AWS Region Context** | `us-east-1` (Note: IAM is a global service) |
| **Method** | AWS CLI & JSON Trust Policy Document |

---

## 🎯 Objective

Provision an IAM role combining a Trust Policy ("Who can assume me") with a Permissions Policy ("What can I do"):

```text
Define EC2 Trust Document ──► aws iam create-role ──► Attach iampolicy_ammar ──► Verify Trust & Permissions
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

## 1. Create the EC2 Trust Policy Document

Define the Trust Policy JSON document allowing the EC2 service to issue `sts:AssumeRole` API calls:

```bash
cat << 'EOF' > ec2-trust-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF
```

---

## 2. Create the IAM Role

Execute the role creation command referencing the trust policy document:

```bash
aws iam create-role \
  --role-name iamrole_ammar \
  --assume-role-policy-document file://ec2-trust-policy.json
```

**Example JSON response:**
```json
{
    "Role": {
        "Path": "/",
        "RoleName": "iamrole_ammar",
        "RoleId": "AROAEXAMPLE123456",
        "Arn": "arn:aws:iam::123456789012:role/iamrole_ammar",
        "CreateDate": "2026-08-10T20:30:00Z",
        "AssumeRolePolicyDocument": {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Effect": "Allow",
                    "Principal": {
                        "Service": "ec2.amazonaws.com"
                    },
                    "Action": "sts:AssumeRole"
                }
            ]
        }
    }
}
```

---

## 3. Resolve Policy ARN & Attach Permission Policy

Dynamically resolve the ARN for existing customer-managed policy `iampolicy_ammar` and attach it to `iamrole_ammar`:

```bash
POLICY_ARN=$(aws iam list-policies \
  --scope Local \
  --query "Policies[?PolicyName=='iampolicy_ammar'].Arn" \
  --output text)

aws iam attach-role-policy \
  --role-name iamrole_ammar \
  --policy-arn "$POLICY_ARN"
```

*(Note: The AWS CLI produces no stdout payload upon successful policy attachment.)*

---

## 4. Verify Role Metadata & Trust Relationship

Confirm that the role exists and that the trusted principal is `ec2.amazonaws.com`:

```bash
aws iam get-role \
  --role-name iamrole_ammar \
  --query "Role.{Name:RoleName,Arn:Arn,TrustedService:AssumeRolePolicyDocument.Statement[0].Principal.Service}" \
  --output table
```

**Expected output:**
```text
---------------------------------------------------------------------------------------------
|                                          GetRole                                          |
+------------------------------------------------+-----------------+------------------------+
|                      Arn                       |      Name       |     TrustedService     |
+------------------------------------------------+-----------------+------------------------+
|  arn:aws:iam::123456789012:role/iamrole_ammar  |  iamrole_ammar  |  ec2.amazonaws.com     |
+------------------------------------------------+-----------------+------------------------+
```

---

## 5. Verify Attached Permission Policies

List all managed policies directly attached to `iamrole_ammar`:

```bash
aws iam list-attached-role-policies \
  --role-name iamrole_ammar \
  --query "AttachedPolicies[?PolicyName=='iampolicy_ammar'].{Name:PolicyName,Arn:PolicyArn}" \
  --output table
```

**Expected output:**
```text
--------------------------------------------------------------------------------------------------
|                                    ListAttachedRolePolicies                                    |
+----------------------------------------------------+-------------------------------------------+
|                        Arn                         |                   Name                    |
+----------------------------------------------------+-------------------------------------------+
|  arn:aws:iam::123456789012:policy/iampolicy_ammar  |  iampolicy_ammar                          |
+----------------------------------------------------+-------------------------------------------+
```

---

## 🔍 Complete Verification

Perform an all-in-one execution checking role existence, trusted service, and attached policies:

```bash
echo "Role Name: $(aws iam get-role --role-name iamrole_ammar --query 'Role.RoleName' --output text)"
echo "Trusted Principal: $(aws iam get-role --role-name iamrole_ammar --query 'Role.AssumeRolePolicyDocument.Statement[0].Principal.Service' --output text)"
echo "Attached Policy: $(aws iam list-attached-role-policies --role-name iamrole_ammar --query 'AttachedPolicies[0].PolicyName' --output text)"
```

**Expected output:**
```text
Role Name: iamrole_ammar
Trusted Principal: ec2.amazonaws.com
Attached Policy: iampolicy_ammar
```

---

## 🧠 Key Concepts & Takeaways

| Concept | Description |
| :--- | :--- |
| **Trust Policy vs. Permission Policy** | A **Trust Policy** specifies *who* can assume the role (e.g., `ec2.amazonaws.com` via `sts:AssumeRole`). A **Permissions Policy** defines *what* actions the identity can execute once assumed. Both are required for functional access. |
| **Security via IAM Roles** | IAM Roles supply temporary, short-lived credentials managed and rotated automatically by AWS (via the EC2 Metadata Service / IMDS), eliminating the need for hardcoded AWS credentials in applications. |
| **Instance Profile Note** | To attach an IAM role to a physical or virtual EC2 instance via CLI, an **IAM Instance Profile** must be created and linked to the role (`aws iam create-instance-profile` and `aws iam add-role-to-instance-profile`). |
| **Principle of Least Privilege** | Granting targeted roles to compute instances minimizes attack surface and cost risks in the event of an application-level compromise. |

> *This was Task 20 of 50 AWS tasks in the AWS phase of my KodeKloud 100 Days of Cloud Challenge.*
