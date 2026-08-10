# AWS Task 10 — Attach an Elastic IP to an EC2 Instance

## 📌 Task Overview

In this task, the Nautilus DevOps team required an existing Elastic IP address in the `us-east-1` region to be attached to an existing EC2 instance to ensure a static public IP endpoint. Using the AWS CLI, the Elastic IP named `xfusion-ec2-eip` was located, mapped to its underlying allocation ID, and associated with the EC2 instance named `xfusion-ec2`.

| Requirement | Value |
| :--- | :--- |
| **EC2 Instance** | `xfusion-ec2` |
| **Elastic IP Name** | `xfusion-ec2-eip` |
| **AWS Region** | `us-east-1` |
| **Action** | Associate Elastic IP (`associate-address`) |
| **Method** | AWS CLI |

---

## 🎯 Objective

Establish an association between the static Elastic IP and the targeted EC2 instance:

```text
Elastic IP (xfusion-ec2-eip) ──► Associate ──► EC2 Instance (xfusion-ec2)
```

---

## 🛠️ Prerequisites

Check that the AWS CLI is installed and responsive:
```bash
aws --version
```

Retrieve temporary lab credentials:
```bash
showcreds
```

Verify the active AWS identity:
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

Locate the instance ID using the `Name` tag filter:
```bash
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=xfusion-ec2" \
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

Verify that the target instance is active:
```bash
aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query "Reservations[].Instances[].{Name:Tags[?Key=='Name']|[0].Value,InstanceId:InstanceId,State:State.Name,PrivateIP:PrivateIpAddress,PublicIP:PublicIpAddress}" \
  --output table
```

---

## 2. Locate the Elastic IP Allocation ID

Search for the Elastic IP using its `Name` tag filter:
```bash
aws ec2 describe-addresses \
  --filters "Name=tag:Name,Values=xfusion-ec2-eip" \
  --query "Addresses[].{AllocationId:AllocationId,PublicIp:PublicIp,InstanceId:InstanceId}" \
  --output table
```

**Example output:**
```text
----------------------------------------------------------------
|                       DescribeAddresses                      |
+----------------------+----------------------+----------------+
| AllocationId        | PublicIp             | InstanceId     |
+----------------------+----------------------+----------------+
| eipalloc-012345678   | 18.x.x.x             | None           |
+----------------------+----------------------+----------------+
```

Save the allocation ID:
```bash
ALLOCATION_ID=eipalloc-0123456789abcdef0
```

> **⚠️ Troubleshooting — Tag Search Returns Empty:** If filtering by `tag:Name` yields no output, the Elastic IP may exist without an explicit `Name` tag. List all addresses in the region to inspect unassociated IPs directly:
> ```bash
> aws ec2 describe-addresses \
>   --query "Addresses[].{AllocationId:AllocationId,PublicIp:PublicIp,InstanceId:InstanceId}" \
>   --output table
> ```

---

## 3. Associate the Elastic IP to the EC2 Instance

Execute the association using both the `INSTANCE_ID` and `ALLOCATION_ID`:
```bash
aws ec2 associate-address \
  --instance-id "$INSTANCE_ID" \
  --allocation-id "$ALLOCATION_ID"
```

**Example JSON response:**
```json
{
    "AssociationId": "eipassoc-0123456789abcdef0"
}
```

---

## 4. Verify the Elastic IP Association

Confirm that the Elastic IP now references the target EC2 instance:
```bash
aws ec2 describe-addresses \
  --allocation-ids "$ALLOCATION_ID" \
  --query "Addresses[].{AllocationId:AllocationId,PublicIp:PublicIp,InstanceId:InstanceId,AssociationId:AssociationId}" \
  --output table
```

Confirm that the EC2 instance reflects the new static public IP:
```bash
aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query "Reservations[].Instances[].{Name:Tags[?Key=='Name']|[0].Value,InstanceId:InstanceId,PublicIP:PublicIpAddress,State:State.Name}" \
  --output table
```

---

## 🔍 Complete Verification

Run an end-to-end check comparing address allocation details with the instance configuration:

```bash
# Verify address mapping
aws ec2 describe-addresses \
  --allocation-ids "$ALLOCATION_ID" \
  --query "Addresses[].{PublicIP:PublicIp,InstanceId:InstanceId,AssociationId:AssociationId}" \
  --output table

# Verify instance public endpoint
aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query "Reservations[].Instances[].{Name:Tags[?Key=='Name']|[0].Value,InstanceId:InstanceId,PublicIP:PublicIpAddress,State:State.Name}" \
  --output table
```

---

## 🔄 Public IPv4 vs. Elastic IP

| Feature | Dynamic Public IPv4 | Elastic IP (EIP) |
| :--- | :--- | :--- |
| **Persistence** | Changes when stopped/started | Persistent static IP allocated to account |
| **Reassociation** | Tied to specific instance network interface | Can be remapped dynamically to replacement instances |
| **Use Case** | Ephemeral or temporary workloads | Critical endpoints requiring persistent DNS/A-records |

---

## 🛡️ Security & Architecture Best Practices

* **Security Groups:** Assigning a static public IP increases exposure. Strict ingress rules must be applied on the attached Security Group (e.g., restricting SSH port 22 to authorized source IPs/CIDRs rather than `0.0.0.0/0`).
* **Private Subnet Architecture:** Production web applications should ideally run in private subnets behind a public Load Balancer (ALB/NLB) rather than directly assigning Elastic IPs to individual backend instances.

---

## 🧠 What I Learned & Key Takeaways

The complete CLI workflow for mapping static network infrastructure relies on mapping unique identifiers:

```text
Find Instance ID ──► Find EIP Allocation ID ──► Associate Address ──► Verify Endpoint
```

* **ID-Based Operations:** AWS CLI commands require underlying resource IDs (`AllocationId`, `InstanceId`) rather than display names (`Name` tags).
* **Troubleshooting Unfiltered Queries:** When a tagged search yields no results, running an unfiltered `describe-addresses` prevents assuming a resource is missing when it is merely untagged.

> *This was Task 10 of 50 AWS tasks in the AWS phase of my KodeKloud 100 Days of Cloud Challenge.*
