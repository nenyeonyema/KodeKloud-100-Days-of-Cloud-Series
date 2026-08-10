# AWS Task 3 — Create a Subnet in the Default VPC

## 📌 Task Overview

As part of the Nautilus DevOps team's gradual migration to AWS, the next step was to create a subnet within the default VPC. For this task, a subnet named `xfusion-subnet` was created under the default VPC in the `us-east-1` region using the AWS CLI.

| Requirement | Value |
| :--- | :--- |
| **Subnet Name** | `xfusion-subnet` |
| **VPC** | Default VPC |
| **AWS Region** | `us-east-1` |
| **Method** | AWS CLI |

---

## 🎯 Objective

Create a subnet named `xfusion-subnet` inside the default VPC. The subnet must use a CIDR block that falls within the CIDR range of the default VPC.

---

## 🛠️ Prerequisites

Make sure the AWS CLI is installed:
```bash
aws --version
```

Retrieve the temporary AWS lab credentials:
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

## 1. Find the Default VPC

First, retrieve the default VPC ID:
```bash
aws ec2 describe-vpcs \
  --filters "Name=isDefault,Values=true" \
  --query "Vpcs[0].VpcId" \
  --output text
```

**Example output:**
```text
vpc-0c824e8a4ab3b0b28
```

Save the VPC ID as an environment variable (replace with your actual VPC ID):
```bash
VPC_ID=vpc-0c824e8a4ab3b0b28
```

---

## 2. Check the VPC CIDR Range

Before creating the subnet, check the CIDR block assigned to the default VPC:
```bash
aws ec2 describe-vpcs \
  --vpc-ids "$VPC_ID" \
  --query "Vpcs[].CidrBlock" \
  --output text
```

**Example output:**
```text
172.31.0.0/16
```

> **Note:** The subnet CIDR must be a valid range inside the VPC CIDR. For example, if the VPC is `172.31.0.0/16`, a valid subnet CIDR could be `172.31.1.0/24`.

---

## 3. Check Existing Subnets

Before selecting a CIDR block, check the subnets that already exist in the VPC to prevent overlapping ranges:
```bash
aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query "Subnets[].{SubnetId:SubnetId,CIDR:CidrBlock,AZ:AvailabilityZone,Name:Tags[?Key=='Name']|[0].Value}" \
  --output table
```

---

## 4. Find an Availability Zone

List the available Availability Zones in the region:
```bash
aws ec2 describe-availability-zones \
  --query "AvailabilityZones[?State=='available'].ZoneName" \
  --output text
```

**Example output:**
```text
us-east-1a    us-east-1b    us-east-1c
```

Select an Availability Zone:
```bash
AZ=us-east-1a
```

---

## 5. Create the Subnet

Use a non-overlapping CIDR block within the VPC range:
```bash
aws ec2 create-subnet \
  --vpc-id "$VPC_ID" \
  --cidr-block 172.31.1.0/24 \
  --availability-zone "$AZ" \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=xfusion-subnet}]'
```

**Example output:**
```json
{
    "Subnet": {
        "SubnetId": "subnet-0123456789abcdef0",
        "VpcId": "vpc-0c824e8a4ab3b0b28",
        "CidrBlock": "172.31.1.0/24",
        "AvailabilityZone": "us-east-1a"
    }
}
```

### ⚠️ Important: CIDR Must Be Within the VPC
A subnet cannot use an arbitrary CIDR block. For instance, if the VPC is `172.31.0.0/16`, using `10.0.1.0/24` will fail with:
```text
InvalidSubnet.Range: The CIDR '10.0.1.0/24' is invalid.
```

---

## 6. Verify the Subnet

Retrieve the details of the newly created subnet:
```bash
aws ec2 describe-subnets \
  --filters "Name=tag:Name,Values=xfusion-subnet" \
  --query "Subnets[].{SubnetId:SubnetId,Name:Tags[?Key=='Name']|[0].Value,VPC:VpcId,CIDR:CidrBlock,AZ:AvailabilityZone,State:State}" \
  --output table
```

---

## 7. Verify Subnet CIDR and VPC Association

Confirm the CIDR block independently:
```bash
aws ec2 describe-subnets \
  --filters "Name=tag:Name,Values=xfusion-subnet" \
  --query "Subnets[].CidrBlock" \
  --output text
```

Confirm the VPC association:
```bash
aws ec2 describe-subnets \
  --filters "Name=tag:Name,Values=xfusion-subnet" \
  --query "Subnets[].VpcId" \
  --output text
```

---

## ✅ Final Result

The required subnet was successfully created.

| Configuration | Result |
| :--- | :--- |
| **Subnet Name** | `xfusion-subnet` |
| **VPC** | Default VPC |
| **Region** | `us-east-1` |
| **CIDR** | Valid range within VPC CIDR (`172.31.1.0/24`) |
| **Availability Zone** | `us-east-1a` |
| **State** | `available` |
| **Method** | AWS CLI |

---

## 🧠 What I Learned

* A subnet is a smaller network carved out of a VPC, meaning its CIDR must strictly fit inside the VPC's CIDR range.
* The standard subnet creation workflow is:
  1. Identify Default VPC
  2. Check VPC CIDR
  3. Inspect existing subnets
  4. Select a non-overlapping CIDR range
  5. Create and tag the subnet
  6. Verify state and association

---

## 💡 Key Takeaway

The subnet creation command is straightforward, but selecting the proper CIDR block requires essential networking knowledge. Avoiding CIDR overlaps upfront prevents IP address conflicts and downstream resource deployment failures. Using the AWS CLI makes these configurations repeatable and scriptable.

> *This was Task 3 of 50 AWS tasks in the AWS phase of my KodeKloud 100 Days of Cloud Challenge.*
