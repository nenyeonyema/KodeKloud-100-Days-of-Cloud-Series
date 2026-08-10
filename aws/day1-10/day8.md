# AWS Task 8 — Enable EC2 Stop Protection
# AWS Task 8 — Enable EC2 Stop Protection

## 📌 Task Overview

As part of the AWS migration, the Nautilus DevOps team created an EC2 instance that required additional protection against accidental shutdown operations. For this task, the existing EC2 instance named `nautilus-ec2` in `us-east-1` was located, inspect, and configured with stop protection using the AWS CLI.

| Requirement | Value |
| :--- | :--- |
| **Instance Name** | `nautilus-ec2` |
| **Protection Type** | Stop Protection (`disableApiStop`) |
| **AWS Region** | `us-east-1` |
| **Final Requirement** | Stop Protection Enabled (`True`) |
| **Method** | AWS CLI |

---

## 🎯 Objective

Enable stop protection on the existing EC2 instance `nautilus-ec2` to block standard stop API actions and prevent accidental operational downtime.

> **Key Distinction:** Stop protection (`disableApiStop`) blocks `stop-instances` calls, whereas Termination protection (`disableApiTermination`) blocks `terminate-instances` calls.

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
  --filters "Name=tag:Name,Values=nautilus-ec2" \
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
  --query "Reservations[].Instances[].{Name:Tags[?Key=='Name']|[0].Value,ID:InstanceId,Type:InstanceType,State:State.Name}" \
  --output table
```

Check the current status of the `disableApiStop` attribute prior to modification:
```bash
aws ec2 describe-instance-attribute \
  --instance-id "$INSTANCE_ID" \
  --attribute disableApiStop \
  --query "DisableApiStop.Value" \
  --output text
```

**Expected output (unprotected):**
```text
False
```

---

## 3. Enable Stop Protection

Execute the modification command to apply stop protection:
```bash
aws ec2 modify-instance-attribute \
  --instance-id "$INSTANCE_ID" \
  --disable-api-stop
```

*(Note: `modify-instance-attribute` completes silently upon success without producing output text.)*

---

## 4. Verify Stop Protection

Re-query the `disableApiStop` attribute to confirm the setting applied:
```bash
aws ec2 describe-instance-attribute \
  --instance-id "$INSTANCE_ID" \
  --attribute disableApiStop \
  --query "DisableApiStop.Value" \
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
# Retrieve Instance ID dynamically
INSTANCE_ID=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=nautilus-ec2" \
  --query "Reservations[0].Instances[0].InstanceId" \
  --output text)

# Check Stop Protection setting (Expected: True)
aws ec2 describe-instance-attribute \
  --instance-id "$INSTANCE_ID" \
  --attribute disableApiStop \
  --query "DisableApiStop.Value" \
  --output text
```

Detailed JSON attribute verification:
```bash
aws ec2 describe-instance-attribute \
  --instance-id "$INSTANCE_ID" \
  --attribute disableApiStop
```

**Expected JSON response:**
```json
{
    "InstanceId": "i-0123456789abcdef0",
    "DisableApiStop": {
        "Value": true
    }
}
```

---

## 🛡️ Stop Protection vs. Termination Protection

These two AWS safeguards address distinct operational risks:

| Safeguard | CLI Attribute | Flag / Command Option | Action Blocked |
| :--- | :--- | :--- | :--- |
| **Stop Protection** | `disableApiStop` | `--disable-api-stop` | `aws ec2 stop-instances` |
| **Termination Protection** | `disableApiTermination` | `--disable-api-termination` | `aws ec2 terminate-instances` |

---

## 💰 Operational Considerations

While stop protection prevents accidental downtime for critical production services, it can impact automated cost-management scripts (such as auto-stop schedules for non-production environments). Protection mechanisms should be applied selectively based on workload criticality:

```text
Critical Production Workloads ──► Enable Stop Protection (High Availability)
Non-Production / Dev Environments ──► Keep Disabled (Allows Auto-Stop / Cost Savings)
```

---

## 🧠 What I Learned & Key Takeaways

The complete CLI workflow for modifying instance safety attributes is lightweight and direct:

```text
Find Instance ──► Check Attribute State ──► Modify Attribute ──► Verify Setting
```

* **Targeted Safeguards:** Security and protection flags must match the exact risk (accidental stop vs. accidental termination).
* **Instant Application:** Attribute modifications like `disableApiStop` do not require stopping or restarting the EC2 instance; they take effect immediately on live instances.

> *This was Task 8 of 50 AWS tasks in the AWS phase of my KodeKloud 100 Days of Cloud Challenge.*
