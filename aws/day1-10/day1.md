# AWS Task 1 — Create an RSA Key Pair

## 📌 Task Overview

The Nautilus DevOps team is beginning its migration to AWS by creating the foundational resources required for future infrastructure. For this task, an EC2 key pair was created with the following requirements:

| Requirement | Value |
| :--- | :--- |
| **Key Pair Name** | `devops-kp` |
| **Key Pair Type** | RSA |
| **AWS Region** | `us-east-1` |

The task was completed using the AWS CLI rather than the AWS Management Console.

---

## 🎯 Objective

Create an RSA EC2 key pair named `devops-kp`. The key pair can later be used to securely connect to EC2 instances through SSH.

---

## 🛠️ Prerequisites

Ensure the AWS CLI is installed on the `aws-client` host.

Check the AWS CLI installation:
```bash
aws --version
```

Verify the AWS identity:
```bash
aws sts get-caller-identity
```

Retrieve the temporary lab credentials:
```bash
showcreds
```

Make sure the AWS CLI is configured for `us-east-1`.

---

## 1. Set the AWS Region

Set the default AWS region to `us-east-1`:
```bash
export AWS_DEFAULT_REGION=us-east-1
```

Verify:
```bash
aws configure get region
```

**Expected output:**
```text
us-east-1
```

---

## 2. Create the RSA Key Pair

Create the required RSA key pair:
```bash
aws ec2 create-key-pair \
  --key-name devops-kp \
  --key-type rsa \
  --query "KeyMaterial" \
  --output text > devops-kp.pem
```

### Command Breakdown
* `--key-name devops-kp`: Creates the key pair with the required name.
* `--key-type rsa`: Specifies RSA as the key type.
* `--query "KeyMaterial"`: Extracts the private key.
* `--output text`: Returns the key as plain text.
* `> devops-kp.pem`: Saves the private key locally.

---

## 3. Secure the Private Key

Set restrictive permissions on the private key:
```bash
chmod 400 devops-kp.pem
```

Verify the permissions:
```bash
ls -l devops-kp.pem
```

The file should only be readable by the owner.

**Example output:**
```text
-r-------- 1 user user 1678 Jan 6 11:00 devops-kp.pem
```

---

## 4. Verify the Key Pair

Confirm that AWS successfully created the key pair:
```bash
aws ec2 describe-key-pairs \
  --key-names devops-kp \
  --query "KeyPairs[].{Name:KeyName,Type:KeyType}" \
  --output table
```

**Expected output:**
```text
--------------------------------
|       DescribeKeyPairs       |
+---------------+--------------+
| Name          | Type         |
+---------------+--------------+
| devops-kp     | rsa          |
+---------------+--------------+
```

---

## 5. Check the Key Pair Details

To view the complete key pair information:
```bash
aws ec2 describe-key-pairs \
  --key-names devops-kp
```

This confirms that the key pair exists in the AWS account.

---

## ✅ Result

The required EC2 key pair was successfully created.

| Configuration | Result |
| :--- | :--- |
| **Key Pair Name** | `devops-kp` |
| **Key Pair Type** | `rsa` |
| **Region** | `us-east-1` |
| **Private Key** | `devops-kp.pem` |
| **Creation Method** | AWS CLI |

---

## 🔐 Security Considerations

* The `.pem` file contains the private key and must be protected.
* **Never** commit it to Git or upload it to a public repository.
* Add private key files to `.gitignore`:

```bash
echo "*.pem" >> .gitignore
```

Check Git status:
```bash
git status
```

The private key should remain securely stored on the machine where it is needed.

---

## 🧠 What I Learned

Although creating a key pair is a simple AWS task, it introduced an important EC2 security concept:

* AWS stores the **public portion** of the key pair, while the **private key** is provided to the user when the key pair is created.
* The private key **cannot** be downloaded again from AWS if it is lost.
* This makes secure private-key management an essential part of working with EC2.

---

## 💡 Key Takeaway

This task reinforced using the AWS CLI during the **KodeKloud 100 Days of Cloud Challenge**. Instead of navigating through multiple console screens, the resource was created and verified with a few commands.

Using the CLI makes cloud operations:
* Repeatable
* Scriptable
* Faster
* Easier to automate
* Less dependent on manual clicks

This was Task 1 of 50 AWS tasks in my KodeKloud 100 Days of Cloud Challenge.
