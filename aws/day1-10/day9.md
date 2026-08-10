# AWS Task 9 — Enable EC2 Termination Protection

## 📌 Task Overview

During the AWS migration, the Nautilus DevOps team identified an EC2 instance that required protection against accidental deletion or termination. For this task, the existing EC2 instance named `devops-ec2` in `us-east-1` was located, inspected, and configured with termination protection using the AWS CLI.

| Requirement | Value |
| :--- | :--- |
| **Instance Name** | `devops-ec2` |
| **Protection Type** | Termination Protection (`disableApiTermination`) |
| **AWS Region** | `us-east-1` |
| **Final Requirement** | Termination Protection Enabled (`True`) |
| **Method** | AWS CLI |

---

## 🎯 Objective

Enable termination protection on the existing EC2 instance `devops-ec2` to block API/Console termination calls and prevent permanent data loss or service disruption.

> **Key Distinction:** Termination protection (`disableApiTermination`) prevents permanent deletion via `terminate-instances`, while Stop protection (`disableApiStop`) prevents shutdown via `stop-instances`.

---

## 🛠️ Prerequisites

Check that the AWS CLI is available:
```bash
aws --version
```

Retrieve temporary lab credentials:
```bash
showcreds
```

Verify the AWS identity:
```bash
aws sts get-caller-identity
```

Set and verify the target region:
```bash
export AWS_DEFAULT_REGION=us-east-1
aws configure get region
```

**Expected output:**
```text
us-east-1
```

---

## 1. Find the EC2 Instance

Query the instance ID using the `Name` tag filter:
```bash
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=devops-ec2" \
  --query "Reservations[].Instances[].InstanceId" \
  --output text
```

**Example output:**
```text
i-0123456789abcdef0
```

Save the instance ID as an environment variable:
```bash
INSTANCE_ID=i-0123456789abcdef0
```

---

## 2. Verify Instance Details & Current Protection State

Verify that the target instance is running and identified properly:
```bash
aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query "Reservations[].Instances[].{Name:Tags[?Key=='Name']|[0].Value,InstanceId:InstanceId,Type:InstanceType,State:State.Name}" \
  --output table
```

Check the current status of the `disableApiTermination` attribute prior to modification:
```bash
aws ec2 describe-instance-attribute \
  --instance-id "$INSTANCE_ID" \
  --attribute disableApiTermination \
  --query "DisableApiTermination.Value" \
  --output text
```

**Expected output (unprotected):**
```text
False
```

---

## 3. Enable Termination Protection

Execute the modification command to apply termination protection:
```bash
aws ec2 modify-instance-attribute \
  --instance-id "$INSTANCE_ID" \
  --disable-api-termination
```

*(Note: `modify-instance-attribute` completes silently upon success without producing output text.)*

---

## 4. Verify Termination Protection

Re-query the `disableApiTermination` attribute to confirm the setting applied:
```bash
aws ec2 describe-instance-attribute \
  --instance-id "$INSTANCE_ID" \
  --attribute disableApiTermination \
  --query "DisableApiTermination.Value" \
  --output text
```

**Expected output:**
```text
True
```

---

## 🔍 Complete Verification

Verify both instance identity and protection state end-to-end:

```bash
# Retrieve Instance ID dynamically and inspect attribute
INSTANCE_ID=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=devops-ec2" \
  --query "Reservations[0].Instances[0].InstanceId" \
  --output text)

echo "Instance ID: $INSTANCE_ID"

aws ec2 describe-instance-attribute \
  --instance-id "$INSTANCE_ID" \
  --attribute disableApiTermination \
  --query "DisableApiTermination.Value" \
  --output text
```

Detailed JSON attribute verification:
```bash
aws ec2 describe-instance-attribute \
  --instance-id "$INSTANCE_ID" \
  --attribute disableApiTermination
```

**Expected JSON response:**
```json
{
    "InstanceId": "i-0123456789abcdef0",
    "DisableApiTermination": {
        "Value": true
    }
}
```

---

## 🛡️ Termination Protection vs. Stop Protection

These two AWS safeguards address distinct operational risks:

| Safeguard | CLI Attribute | Flag / Command Option | Action Blocked |
| :--- | :--- | :--- | :--- |
| **Termination Protection** | `disableApiTermination` | `--disable-api-termination` | `aws ec2 terminate-instances` |
| **Stop Protection** | `disableApiStop` | `--disable-api-stop` | `aws ec2 stop-instances` |

---

## 💰 Operational Considerations

Termination protection prevents accidental instance deletion and ephemeral storage loss for critical services. However, it requires active governance during resource cleanup:

```text
Active Workload ──► Enable Protection (Prevents Accidental Deletion)
Decommission Stage ──► Disable Protection via API/Console ──► Terminate Instance
```

Protection should guard against operational mistakes without impeding intentional lifecycle management and resource inventory cleanup.

---

## 🧠 What I Learned & Key Takeaways

The CLI workflow for safety modifications is repeatable and non-disruptive:

```text
Find Instance ──► Inspect Attribute ──► Modify Attribute ──► Verify Setting
```

* **Immediate Application:** Modifying `disableApiTermination` takes effect live on the instance without requiring a reboot or stop sequence.
* **Targeted Controls:** Matching specific flags to operational risks ensures critical production servers remain protected while allowing necessary administrative tasks when executed deliberately.

> *This was Task 9 of 50 AWS tasks in the AWS phase of my KodeKloud 100 Days of Cloud Challenge.*
