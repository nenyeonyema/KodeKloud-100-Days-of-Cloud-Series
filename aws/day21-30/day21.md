# AWS Task 21 — EC2 Instance with Elastic IP

## 📌 Task Overview

In this task, the Nautilus DevOps team required a dedicated EC2 instance for hosting application services with a persistent public IPv4 address. Standard EC2 public IP addresses are dynamic and change whenever an instance is stopped and restarted. To guarantee network stability, an Elastic IP (EIP) was allocated, named `devops-eip`, and associated directly with the target instance `devops-ec2`.

| Requirement | Value |
| :--- | :--- |
| **EC2 Instance Name** | `devops-ec2` |
| **AMI Base** | Ubuntu 22.04 LTS (`x86_64` HVM) |
| **Instance Type** | `t2.micro` |
| **Elastic IP Name** | `devops-eip` |
| **AWS Region Context** | `us-east-1` |
| **Method** | AWS CLI |

---

## 🎯 Objective

Provision a Linux compute instance, allocate a persistent IPv4 address, and establish the association:

```text
Find Ubuntu 22.04 AMI ──► Launch devops-ec2 ──► Allocate devops-eip ──► Associate EIP to Instance ──► Verify Public IP Continuity
```

---

## 🛠️ Prerequisites

Set the default region and query the default VPC and subnet infrastructure:

```bash
export AWS_DEFAULT_REGION=us-east-1

# Resolve network topology
VPC_ID=$(aws ec2 describe-vpcs --filters "Name=is-default,Values=true" --query "Vpcs[0].VpcId" --output text)
SUBNET_ID=$(aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID" --query "Subnets[0].SubnetId" --output text)
SG_ID=$(aws ec2 describe-security-groups --filters "Name=vpc-id,Values=$VPC_ID" "Name=group-name,Values=default" --query "SecurityGroups[0].GroupId" --output text)
```

---

## 1. Resolve Latest Ubuntu 22.04 AMI

Retrieve the dynamic AMI ID for the latest official Ubuntu 22.04 LTS release in `us-east-1`:

```bash
AMI_ID=$(aws ec2 describe-images \
  --owners 099720109477 \
  --filters \
    "Name=name,Values=ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*" \
    "Name=state,Values=available" \
  --query "Images | sort_by(@, &CreationDate)[-1].ImageId" \
  --output text)

echo "Resolved AMI ID: $AMI_ID"
```

---

## 2. Launch the EC2 Instance

Launch `devops-ec2` inside the default subnet and tag the resource upon creation:

```bash
INSTANCE_ID=$(aws ec2 run-instances \
  --image-id "$AMI_ID" \
  --instance-type t2.micro \
  --subnet-id "$SUBNET_ID" \
  --security-group-ids "$SG_ID" \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=devops-ec2}]' \
  --query "Instances[0].InstanceId" \
  --output text)

echo "Target Instance ID: $INSTANCE_ID"
```

Wait until the instance transitions to the `running` state:

```bash
aws ec2 wait instance-running --instance-ids "$INSTANCE_ID"
```

---

## 3. Allocate and Tag the Elastic IP

Allocate a static public IPv4 address from Amazon's pool and tag it as `devops-eip`:

```bash
EIP_JSON=$(aws ec2 allocate-address --domain vpc --output json)

ALLOCATION_ID=$(echo "$EIP_JSON" | jq -r '.AllocationId')
PUBLIC_IP=$(echo "$EIP_JSON" | jq -r '.PublicIp')

aws ec2 create-tags \
  --resources "$ALLOCATION_ID" \
  --tags Key=Name,Value=devops-eip
```

---

## 4. Associate Elastic IP with EC2 Instance

Bind the allocated Elastic IP address to `devops-ec2`:

```bash
ASSOCIATION_ID=$(aws ec2 associate-address \
  --instance-id "$INSTANCE_ID" \
  --allocation-id "$ALLOCATION_ID" \
  --query "AssociationId" \
  --output text)

echo "Established Association ID: $ASSOCIATION_ID"
```

---

## 🔍 Complete Verification

Verify the complete association lifecycle between `devops-ec2` and `devops-eip`:

```bash
aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query "Reservations[].Instances[].{Name:Tags[?Key=='Name']|[0].Value,InstanceId:InstanceId,State:State.Name,PrivateIP:PrivateIpAddress,PublicIP:PublicIpAddress}" \
  --output table
```

**Expected output:**
```text
------------------------------------------------------------------------------------------------------
|                                          DescribeInstances                                         |
+-------------------+----------------------+----------------+-------------------+--------------------+
|    InstanceId     |        Name          |   PrivateIP    |     PublicIP      |       State        |
+-------------------+----------------------+----------------+-------------------+--------------------+
|  i-0123456789abc  |  devops-ec2          |  172.31.x.x    |  44.x.x.x         |  running           |
+-------------------+----------------------+----------------+-------------------+--------------------+
```

Verify the Elastic IP allocation details:

```bash
aws ec2 describe-addresses \
  --allocation-ids "$ALLOCATION_ID" \
  --query "Addresses[0].{Name:Tags[?Key=='Name']|[0].Value,PublicIP:PublicIp,InstanceId:InstanceId,AllocationId:AllocationId}" \
  --output table
```

---

## 🧠 Key Concepts & Takeaways

| Concept | Description |
| :--- | :--- |
| **Dynamic vs. Elastic IPs** | Default EC2 auto-assigned public IPv4 addresses release automatically when an instance stops. An **Elastic IP (EIP)** is a static public IPv4 address attached to an AWS account that persists across instance stops and starts. |
| **Billing Mechanism** | To prevent IPv4 address space hoarding, AWS charges for Elastic IPs when they are allocated but **not** associated with a running instance or when associated with stopped instances/unattached network interfaces. |
| **Instance Reassociation** | If an instance with a standard public IP has an Elastic IP associated with it, its original auto-assigned public IP is released back into Amazon's public IP pool. |
| **Primary Use Cases** | Essential for hosting public web applications, configuring external DNS A records, whitelisting static outbound IP addresses in firewall rules, and maintaining service availability during instance replacement. |

> *This was Task 21 of 50 AWS tasks in the AWS phase of my KodeKloud 100 Days of Cloud Challenge.*
