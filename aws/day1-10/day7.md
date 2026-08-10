# AWS Task 7 — Change EC2 Instance Type

## 📌 Task Overview

During the migration process, the Nautilus DevOps team identified an EC2 instance that was underutilized. To optimize resource usage, they decided to change its instance type from `t2.micro` to the smaller `t2.nano`. For this task, the instance named `xfusion-ec2` was stopped, modified, restarted, and verified using the AWS CLI.

| Requirement | Value |
| :--- | :--- |
| **Instance Name** | `xfusion-ec2` |
| **Current Type** | `t2.micro` |
| **New Type** | `t2.nano` |
| **AWS Region** | `us-east-1` |
| **Final State** | `running` |
| **Method** | AWS CLI |

---

## 🎯 Objective

Change `xfusion-ec2` from `t2.micro` to `t2.nano` and ensure the instance is running with healthy status checks after the modification.

---

## 🛠️ Prerequisites

Check the AWS CLI:
```bash
aws --version
```

Retrieve the temporary lab credentials:
```bash
showcreds
```

Verify the AWS identity:
```bash
aws sts get-caller-identity
```

Set and verify the required region:
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

Retrieve the instance ID using the `Name` tag filter:
```bash
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=xfusion-ec2" \
  --query "Reservations[].Instances[].InstanceId" \
  --output text
```

**Example output:**
```text
i-0524a12098f5a0d71
```

Save the returned instance ID as an environment variable:
```bash
INSTANCE_ID=i-0524a12098f5a0d71
```

---

## 2. Check Instance State & Health Status

Check the current state and instance type:
```bash
aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query "Reservations[].Instances[].{ID:InstanceId,Type:InstanceType,State:State.Name}" \
  --output table
```

Confirm that the status checks are complete before making changes:
```bash
aws ec2 describe-instance-status \
  --instance-ids "$INSTANCE_ID" \
  --query "InstanceStatuses[].{InstanceState:InstanceState.Name,System:SystemStatus.Status,Instance:InstanceStatus.Status}" \
  --output table
```

**Expected output:**
```text
------------------------------------------------------
|              DescribeInstanceStatus                |
+---------------+----------+--------------------------+
| InstanceState | System   | Instance                 |
+---------------+----------+--------------------------+
| running       | ok       | ok                       |
+---------------+----------+--------------------------+
```

> **⚠️ Troubleshooting — Malformed Instance ID:** Always copy the complete string returned by `describe-instances`. Truncating an ID causes AWS to throw `InvalidInstanceID.Malformed`.

---

## 3. Stop the Instance

Changing an EC2 instance type requires the instance to be in a `stopped` state:
```bash
aws ec2 stop-instances \
  --instance-ids "$INSTANCE_ID"
```

Wait until the instance has completely stopped:
```bash
aws ec2 wait instance-stopped \
  --instance-ids "$INSTANCE_ID"
```

Verify the stopped state:
```bash
aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query "Reservations[].Instances[].State.Name" \
  --output text
```

**Expected output:**
```text
stopped
```

---

## 4. Change the Instance Type

Modify the attribute to set the type to `t2.nano`:
```bash
aws ec2 modify-instance-attribute \
  --instance-id "$INSTANCE_ID" \
  --instance-type '{"Value":"t2.nano"}'
```

> **⚠️ Troubleshooting — Syntax Error:** In Bash multi-line commands, ensure there are no trailing spaces after the trailing backslash `\`. Trailing spaces break line continuation. Alternatively, run as a single line:
> ```bash
> aws ec2 modify-instance-attribute --instance-id "$INSTANCE_ID" --instance-type '{"Value":"t2.nano"}'
> ```

---

## 5. Verify New Type & Restart Instance

Confirm the attribute updated while stopped:
```bash
aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query "Reservations[].Instances[].{Name:Tags[?Key=='Name']|[0].Value,Type:InstanceType,State:State.Name}" \
  --output table
```

Start the instance back up:
```bash
aws ec2 start-instances \
  --instance-ids "$INSTANCE_ID"
```

Wait until the instance transitions to `running`:
```bash
aws ec2 wait instance-running \
  --instance-ids "$INSTANCE_ID"
```

Wait for status checks to pass:
```bash
aws ec2 wait instance-status-ok \
  --instance-ids "$INSTANCE_ID"
```

---

## 🔍 Final Verification

Run the following checks prior to task submission:

```bash
# Verify instance type (Expected: t2.nano)
aws ec2 describe-instances --instance-ids "$INSTANCE_ID" --query "Reservations[].Instances[].InstanceType" --output text

# Verify instance state (Expected: running)
aws ec2 describe-instances --instance-ids "$INSTANCE_ID" --query "Reservations[].Instances[].State.Name" --output text

# Verify status checks (Expected: ok / ok)
aws ec2 describe-instance-status --instance-ids "$INSTANCE_ID" --query "InstanceStatuses[].{System:SystemStatus.Status,Instance:InstanceStatus.Status}" --output table
```

---

## 📊 Before and After Summary

| Configuration | Before | After |
| :--- | :--- | :--- |
| **Instance Name** | `xfusion-ec2` | `xfusion-ec2` |
| **Instance Type** | `t2.micro` | `t2.nano` |
| **State** | `running` | `running` |
| **Status Checks** | Completed (`ok`) | Completed (`ok`) |
| **Region** | `us-east-1` | `us-east-1` |

---

## 💰 Cost Optimization & Right-Sizing

Right-sizing EC2 instances is a core FinOps practice in cloud management:

```text
Monitor Workload ──► Identify Underutilization ──► Right-Size Instance ──► Reduce Unused Capacity ──► Lower Costs
```

* Moving from `t2.micro` to `t2.nano` reduces allocated vCPU and RAM, decreasing hourly resource costs.
* Modifications must be validated against performance metrics (CPU, memory, IOPS) to ensure the smaller instance type provides sufficient capacity for the workload.

---

## 🧠 What I Learned & Key Takeaways

The complete lifecycle for modifying an EBS-backed EC2 instance type follows a strict linear sequence:

```text
Inspect Health ──► Stop Instance ──► Modify Attribute ──► Start Instance ──► Verify Status
```

* **Instance Modification Limits:** EBS-backed instances must be stopped before changing instance types.
* **CLI Precision:** Command-line syntax (e.g., proper JSON string parameters and escaping backslashes) is critical when updating instance attributes via script.
* **Health Validation:** Always wait for `instance-status-ok` after restarting to ensure network reachability and underlying hypervisor health.

> *This was Task 7 of 50 AWS tasks in the AWS phase of my KodeKloud 100 Days of Cloud Challenge.*
