# AWS Task 14 — Terminate an EC2 Instance

## 📌 Task Overview

In this task, the Nautilus DevOps team identified an obsolete EC2 instance (`datacenter-ec2`) following a infrastructure migration. The objective was to safely remove the unused compute instance in `us-east-1` via the AWS CLI, handling API termination protection if present, and validating that the resource reached the `terminated` state before task submission.

| Requirement | Value |
| :--- | :--- |
| **EC2 Instance** | `datacenter-ec2` |
| **AWS Region** | `us-east-1` |
| **Action** | Terminate instance (`terminate-instances`) |
| **Target Final State** | `terminated` |
| **Method** | AWS CLI |

---

## 🎯 Objective

Safely lifecycle and permanently decommission an obsolete EC2 instance:

```text
EC2 Instance (datacenter-ec2) ──► Disable Protection (if set) ──► terminate-instances ──► shutting-down ──► terminated
```

---

## 🛠️ Prerequisites

Set the active region for the AWS CLI session:
```bash
export AWS_DEFAULT_REGION=us-east-1
aws configure get region
```

**Expected output:**
```text
us-east-1
```

---

## 1. Retrieve and Verify the EC2 Instance

Locate the target instance ID using the `Name` tag filter:
```bash
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=datacenter-ec2" \
  --query "Reservations[].Instances[].InstanceId" \
  --output text
```

**Example output:**
```text
i-0123456789abcdef0
```

Save the instance ID:
```bash
INSTANCE_ID=i-0123456789abcdef0
```

Verify its current state (`running` or `stopped`):
```bash
aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query "Reservations[].Instances[].State.Name" \
  --output text
```

---

## 2. Check and Disable Termination Protection

Before attempting termination, check if API termination protection (`disableApiTermination`) is active:

```bash
aws ec2 describe-instance-attribute \
  --instance-id "$INSTANCE_ID" \
  --attribute disableApiTermination \
  --query "DisableApiTermination.Value" \
  --output text
```

If the output returns `True`, disable termination protection:
```bash
aws ec2 modify-instance-attribute \
  --instance-id "$INSTANCE_ID" \
  --no-disable-api-termination
```

Re-verify that protection is now disabled (`False`):
```bash
aws ec2 describe-instance-attribute \
  --instance-id "$INSTANCE_ID" \
  --attribute disableApiTermination \
  --query "DisableApiTermination.Value" \
  --output text
```

---

## 3. Terminate the EC2 Instance

Issue the termination API call:

```bash
aws ec2 terminate-instances \
  --instance-ids "$INSTANCE_ID"
```

**Example JSON response:**
```json
{
    "TerminatingInstances": [
        {
            "InstanceId": "i-0123456789abcdef0",
            "CurrentState": {
                "Code": 32,
                "Name": "shutting-down"
            },
            "PreviousState": {
                "Code": 16,
                "Name": "running"
            }
        }
    ]
}
```

---

## 4. Monitor and Wait for Instance Termination

The initial state transitions to `shutting-down` while AWS tears down instance resources. Block execution until the instance reaches the `terminated` state:

```bash
aws ec2 wait instance-terminated \
  --instance-ids "$INSTANCE_ID"
```

---

## 🔍 Complete Verification

Verify that the instance state reflects `terminated`:

```bash
aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --include-all-instances \
  --query "Reservations[].Instances[].{InstanceId:InstanceId,State:State.Name}" \
  --output table
```

**Expected output:**
```text
----------------------------------------
|           DescribeInstances          |
+----------------------+---------------+
|  InstanceId          |  State        |
+----------------------+---------------+
|  i-0123456789abcdef0 |  terminated   |
+----------------------+---------------+
```

---

## 🧠 Key Concepts & Takeaways

| Concept | Description |
| :--- | :--- |
| **Stop vs. Terminate** | **Stopping** shuts down the OS while preserving instance IDs and root EBS volume configurations; **terminating** permanently deletes the compute instance and its default root volume. |
| **Termination Protection** | An safety attribute (`disableApiTermination`) preventing accidental deletion via API or AWS Management Console. |
| **Asynchronous Lifecycle** | `terminate-instances` triggers a state change to `shutting-down`; use `aws ec2 wait instance-terminated` to block until teardown completes. |
| **Decommissioning Pre-checks** | Always review dependent resources (unattached EBS storage, Elastic IPs, DNS records) before destroying an instance to avoid orphaned charges or broken infrastructure dependencies. |

> *This was Task 14 of 50 AWS tasks in the AWS phase of my KodeKloud 100 Days of Cloud Challenge.*
