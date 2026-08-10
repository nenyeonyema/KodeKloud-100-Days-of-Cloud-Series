# AWS Task 2 — Create and Configure a Security Group

## 📌 Task Overview

The Nautilus DevOps team is gradually migrating infrastructure to AWS. As part of the migration, a security group is required to control inbound network traffic to application servers. For this task, a security group was created under the default VPC with HTTP and SSH access using the AWS CLI.

| Requirement | Value |
| :--- | :--- |
| **Security Group Name** | `nautilus-sg` |
| **Description** | Security group for Nautilus App Servers |
| **VPC** | Default VPC |
| **HTTP** | TCP port 80 from `0.0.0.0/0` |
| **SSH** | TCP port 22 from `0.0.0.0/0` |
| **AWS Region** | `us-east-1` |
| **Method** | AWS CLI |

---

## 🎯 Objective

Create a security group named `nautilus-sg`. The security group must allow inbound:
* HTTP traffic on port 80
* SSH traffic on port 22

Both ports should accept traffic from `0.0.0.0/0`.

---

## 🛠️ Prerequisites

Make sure the AWS CLI is installed:
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

Set and verify the required AWS region:
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

The security group must be created inside the default VPC. Run:
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

Save the returned VPC ID as an environment variable (replace with your actual VPC ID):
```bash
VPC_ID=vpc-0c824e8a4ab3b0b28
```

---

## 2. Create the Security Group

Create `nautilus-sg` inside the default VPC:
```bash
aws ec2 create-security-group \
  --group-name nautilus-sg \
  --description "Security group for Nautilus App Servers" \
  --vpc-id "$VPC_ID"
```

**Example output:**
```json
{
    "GroupId": "sg-0123456789abcdef0"
}
```

Save the returned security group ID:
```bash
SG_ID=sg-0123456789abcdef0
```

---

## 3. Add the HTTP Inbound Rule

Allow HTTP traffic on port 80 from anywhere:
```bash
aws ec2 authorize-security-group-ingress \
  --group-id "$SG_ID" \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0
```

This allows clients on the internet to access applications running over HTTP.

---

## 4. Add the SSH Inbound Rule

Allow SSH traffic on port 22 from anywhere:
```bash
aws ec2 authorize-security-group-ingress \
  --group-id "$SG_ID" \
  --protocol tcp \
  --port 22 \
  --cidr 0.0.0.0/0
```

This allows SSH connections to instances associated with the security group.

> **Production Note:** Opening SSH to `0.0.0.0/0` is generally not recommended. In a real environment, SSH should be restricted to a trusted IP range, VPN, or bastion host. This rule is used because it is specifically required by this lab.

---

## 5. Verify the Security Group

Check that the security group exists and has proper attributes:
```bash
aws ec2 describe-security-groups \
  --group-ids "$SG_ID" \
  --query "SecurityGroups[].{Name:GroupName,Description:Description,VPC:VpcId,GroupId:GroupId}" \
  --output table
```

---

## 6. Verify the Inbound Rules

Check the allowed ingress rules:
```bash
aws ec2 describe-security-groups \
  --group-ids "$SG_ID" \
  --query "SecurityGroups[].IpPermissions[].{Protocol:IpProtocol,FromPort:FromPort,ToPort:ToPort,Source:IpRanges[].CidrIp}" \
  --output table
```

**Expected output:**
```text
-------------------------------------------------
| Protocol | FromPort | ToPort | Source         |
+----------+----------+--------+----------------+
| tcp      | 80       | 80     | 0.0.0.0/0      |
| tcp      | 22       | 22     | 0.0.0.0/0      |
-------------------------------------------------
```

### 🔍 Alternative: Verify Everything in JSON
To inspect all security group details in JSON format:
```bash
aws ec2 describe-security-groups \
  --group-ids "$SG_ID" \
  --output json
```

---

## ✅ Final Result

The required security group was successfully created and configured.

| Configuration | Result |
| :--- | :--- |
| **Security Group** | `nautilus-sg` |
| **Description** | Security group for Nautilus App Servers |
| **VPC** | Default VPC |
| **HTTP Access** | TCP 80 (`0.0.0.0/0`) |
| **SSH Access** | TCP 22 (`0.0.0.0/0`) |
| **Region** | `us-east-1` |
| **Method** | AWS CLI |

---

## 🔐 Security Considerations

Although this lab requires `0.0.0.0/0` for both HTTP and SSH, opening SSH to the entire internet is risky for production environments.

A more secure production configuration restricts SSH to a known single IP (`/32` mask):
```bash
aws ec2 authorize-security-group-ingress \
  --group-id "$SG_ID" \
  --protocol tcp \
  --port 22 \
  --cidr YOUR_PUBLIC_IP/32
```

HTTP (Port 80) usually remains publicly accessible when hosting public web applications.

---

## 🧠 What I Learned

This task reinforced that a security group acts as a virtual firewall for AWS resources such as EC2 instances:
* **Security Group** → Controls stateful network traffic
* **Port 80** → HTTP access
* **Port 22** → SSH access
* **CIDR Block** → Defines allowed source IP addresses

A security group must always belong to a specific VPC and cannot exist independently.

---

## 💡 Key Takeaway

Networking security begins before an application is deployed. A single rule like `TCP 22 -> 0.0.0.0/0` determines whether an instance is exposed globally. Understanding VPCs, security groups, ports, protocols, and CIDR ranges is foundational to building secure AWS infrastructure.

Using the AWS CLI makes these configurations repeatable, scriptable, and easy to automate.

> *This was Task 2 of 50 AWS tasks in the AWS phase of my KodeKloud 100 Days of Cloud Challenge.*
