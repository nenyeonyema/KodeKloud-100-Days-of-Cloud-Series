# AWS Task 11 — Attach an ENI to an EC2 Instance

## 📌 Task Overview

The Nautilus DevOps team has an existing EC2 instance and Elastic Network Interface (ENI) in the `us-east-1` region.

The objective is to attach the existing ENI to the existing EC2 instance and verify that the network interface reaches the **attached** state before submitting the task.

### Requirements

* EC2 instance: `devops-ec2`
* Elastic Network Interface: `devops-eni`
* Region: `us-east-1`
* The EC2 instance must finish initialization before making changes.
* The ENI status must be `attached` before submission.
* Solution performed using the **AWS CLI**.

---

## 🛠️ Prerequisites

Make sure the AWS CLI is configured with the credentials provided for the lab.

```bash
aws configure
```

Set the default region to:

```text
us-east-1
```

You can also confirm the active region with:

```bash
aws configure get region
```

Expected:

```text
us-east-1
```

---

## 1. Find the EC2 Instance ID

The task provides the instance name `devops-ec2`.

Use the following command to retrieve its instance ID:

```bash
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=devops-ec2" \
  --query "Reservations[].Instances[].InstanceId" \
  --output text
```

Example:

```text
i-0123456789abcdef0
```

Save the returned instance ID.

---

## 2. Check the EC2 Instance State

Before attaching the ENI, make sure the instance has completed initialization.

Check its current state:

```bash
aws ec2 describe-instances \
  --instance-ids i-0123456789abcdef0 \
  --query "Reservations[].Instances[].State.Name" \
  --output text
```

Expected:

```text
running
```

---

## 3. Check the Instance Status Checks

The task specifically requires the instance initialization to be completed.

Run:

```bash
aws ec2 describe-instance-status \
  --instance-ids i-0123456789abcdef0 \
  --query "InstanceStatuses[].{Instance:InstanceStatus.Status,System:SystemStatus.Status}" \
  --output table
```

Expected result:

```text
--------------------------------
|      DescribeInstanceStatus |
+----------+-------------------+
| Instance | System            |
+----------+-------------------+
| passed   | passed            |
+----------+-------------------+
```

Both instance and system status checks should show:

```text
passed
```

If the checks are still initializing, wait before proceeding.

You can also use the AWS CLI waiter:

```bash
aws ec2 wait instance-status-ok \
  --instance-ids i-0123456789abcdef0
```

No output means the status checks have successfully passed.

---

## 4. Find the ENI ID

The task provides the ENI name `devops-eni`.

Retrieve its Network Interface ID:

```bash
aws ec2 describe-network-interfaces \
  --filters "Name=tag:Name,Values=devops-eni" \
  --query "NetworkInterfaces[].NetworkInterfaceId" \
  --output text
```

Example:

```text
eni-0123456789abcdef0
```

Save the ENI ID.

---

## 5. Check the ENI Before Attaching

Verify that the ENI is available and not already attached:

```bash
aws ec2 describe-network-interfaces \
  --network-interface-ids eni-0123456789abcdef0 \
  --query "NetworkInterfaces[].{ENI:NetworkInterfaceId,Status:Status,Description:Description}" \
  --output table
```

The ENI should normally show:

```text
available
```

before attaching it.

---

## 6. Attach the ENI to the EC2 Instance

Use the following command:

```bash
aws ec2 attach-network-interface \
  --network-interface-id eni-0123456789abcdef0 \
  --instance-id i-0123456789abcdef0 \
  --device-index 1
```

### Why `--device-index 1`?

The primary network interface of an EC2 instance normally uses device index `0`.

Therefore, the additional ENI should use:

```text
1
```

The command should return an attachment ID similar to:

```text
eni-attach-0123456789abcdef0
```

---

## 7. Verify the ENI Is Attached

This is the most important validation step.

Run:

```bash
aws ec2 describe-network-interfaces \
  --network-interface-ids eni-0123456789abcdef0 \
  --query "NetworkInterfaces[].{ENI:NetworkInterfaceId,Status:Status,Instance:Attachment.InstanceId,Device:Attachment.DeviceIndex}" \
  --output table
```

Expected:

```text
------------------------------------------------
|          DescribeNetworkInterfaces            |
+----------------+------------------------------+
| ENI            | eni-0123456789abcdef0         |
| Status         | in-use                       |
| Instance       | i-0123456789abcdef0          |
| Device         | 1                            |
+----------------+------------------------------+
```

The important values are:

```text
Status: in-use
Instance: i-0123456789abcdef0
Device: 1
```

AWS reports an attached ENI as `in-use`.

---

## 8. Alternative Verification from the EC2 Instance

You can also verify that the ENI is associated with the instance:

```bash
aws ec2 describe-instances \
  --instance-ids i-0123456789abcdef0 \
  --query "Reservations[].Instances[].NetworkInterfaces[].{ENI:NetworkInterfaceId,Status:Status,Device:Attachment.DeviceIndex}" \
  --output table
```

You should see both the primary interface and the newly attached ENI.

---

## ✅ Final Validation

Before submitting the task, confirm:

```bash
aws ec2 describe-instances \
  --instance-ids i-0123456789abcdef0 \
  --query "Reservations[].Instances[].State.Name" \
  --output text
```

Expected:

```text
running
```

Confirm the instance status checks:

```bash
aws ec2 describe-instance-status \
  --instance-ids i-0123456789abcdef0 \
  --query "InstanceStatuses[].{Instance:InstanceStatus.Status,System:SystemStatus.Status}" \
  --output table
```

Both should be:

```text
passed
```

Finally, confirm the ENI:

```bash
aws ec2 describe-network-interfaces \
  --network-interface-ids eni-0123456789abcdef0 \
  --query "NetworkInterfaces[].{Status:Status,Instance:Attachment.InstanceId}" \
  --output table
```

Expected:

```text
Status:   in-use
Instance: i-0123456789abcdef0
```

---

## 🧠 Key Lessons Learned

### 1. An ENI is a separate AWS resource

An Elastic Network Interface can exist independently from an EC2 instance and can be attached to an instance when required.

### 2. Device indexes matter

The primary ENI normally uses:

```text
device-index 0
```

Additional interfaces can use:

```text
device-index 1
```

and higher values.

### 3. Always check instance readiness

AWS tasks may require an instance to complete its status checks before performing infrastructure changes.

A running instance does not necessarily mean that initialization is complete.

### 4. CLI makes the process repeatable

Using the AWS CLI makes it easier to:

* Automate infrastructure operations
* Reduce manual errors
* Verify resource states precisely
* Reuse commands in scripts and CI/CD pipelines

### 5. Always verify the final state

Creating or attaching a resource is only half the task.

The important question is:

> **Did AWS actually reach the state I expected?**

For this task, that means the EC2 instance is running, its status checks have passed, and the ENI is attached to the correct instance.

---

## 🎯 Task Result

**Task 11 completed successfully.**

An existing `devops-eni` Elastic Network Interface was attached to the existing `devops-ec2` EC2 instance in `us-east-1` using the AWS CLI, and the final resource state was verified.

