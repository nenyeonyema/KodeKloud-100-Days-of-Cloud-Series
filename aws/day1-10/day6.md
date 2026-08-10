# AWS Task 6 — Launch an EC2 Instance

## 📌 Task Overview

For this task, an Amazon EC2 instance was launched with specific compute, networking, and authentication requirements. The instance was configured using an Amazon Linux AMI, a `t2.micro` instance type, a newly created RSA key pair, and the default security group using the AWS CLI.

| Requirement | Value |
| :--- | :--- |
| **Instance Name** | `nautilus-ec2` |
| **AMI** | Amazon Linux |
| **Instance Type** | `t2.micro` |
| **Key Pair** | `nautilus-kp` |
| **Key Type** | RSA |
| **Security Group** | Default security group |
| **AWS Region** | `us-east-1` |
| **Method** | AWS CLI |

---

## 🎯 Objective

Launch an EC2 instance named `nautilus-ec2` with:
* Amazon Linux AMI
* `t2.micro` instance type
* RSA key pair named `nautilus-kp`
* Default security group
* Region `us-east-1`

---

## 🛠️ Prerequisites

Check that the AWS CLI is installed:
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

## 1. Find the Default VPC

The instance must use the default security group, so first identify the default VPC:
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

Save the VPC ID as an environment variable:
```bash
VPC_ID=vpc-0c824e8a4ab3b0b28
```

---

## 2. Find the Default Security Group

Retrieve the default security group associated with the default VPC:
```bash
aws ec2 describe-security-groups \
  --filters \
    "Name=vpc-id,Values=$VPC_ID" \
    "Name=group-name,Values=default" \
  --query "SecurityGroups[0].GroupId" \
  --output text
```

**Example output:**
```text
sg-0123456789abcdef0
```

Save the security group ID:
```bash
SG_ID=sg-0123456789abcdef0
```

Verify the security group details:
```bash
aws ec2 describe-security-groups \
  --group-ids "$SG_ID" \
  --query "SecurityGroups[].{Name:GroupName,GroupId:GroupId,VPC:VpcId}" \
  --output table
```

---

## 3. Find an Amazon Linux AMI

Dynamically query the latest available Amazon Linux 2 AMI ID in `us-east-1`:
```bash
aws ec2 describe-images \
  --owners amazon \
  --filters \
    "Name=name,Values=amzn2-ami-hvm-*-x86_64-gp2" \
    "Name=state,Values=available" \
  --query "Images | sort_by(@, &CreationDate) | [-1].{ID:ImageId,Name:Name,CreationDate:CreationDate}" \
  --output table
```

Save the returned AMI ID:
```bash
AMI_ID=ami-xxxxxxxxxxxxxxxxx
```

> **Note:** AMI IDs are region-specific. Always query the AMI within the target region (`us-east-1`) rather than copying an ID from another region.

---

## 4. Create and Secure the RSA Key Pair

Create the required RSA key pair and save the private key locally:
```bash
aws ec2 create-key-pair \
  --key-name nautilus-kp \
  --key-type rsa \
  --query "KeyMaterial" \
  --output text > nautilus-kp.pem
```

Set restrictive permissions on the private key so only the owner can read it:
```bash
chmod 400 nautilus-kp.pem
ls -l nautilus-kp.pem
```

---

## 5. Launch the EC2 Instance

Launch the instance using the retrieved configuration parameters:
```bash
aws ec2 run-instances \
  --image-id "$AMI_ID" \
  --instance-type t2.micro \
  --key-name nautilus-kp \
  --security-group-ids "$SG_ID" \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=nautilus-ec2}]'
```

**Example output:**
```json
{
    "Instances": [
        {
            "InstanceId": "i-0123456789abcdef0",
            "InstanceType": "t2.micro",
            "State": {
                "Name": "pending"
            }
        }
    ]
}
```

Save the instance ID:
```bash
INSTANCE_ID=i-0123456789abcdef0
```

---

## 6. Wait for the Instance to Start

Wait until the instance state transitions from `pending` to `running`:
```bash
aws ec2 wait instance-running \
  --instance-ids "$INSTANCE_ID"
```

---

## 7. Verify the Instance & Components

Verify general instance state and configuration:
```bash
aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query "Reservations[].Instances[].{Name:Tags[?Key=='Name']|[0].Value,InstanceId:InstanceId,Type:InstanceType,State:State.Name,AMI:ImageId,Key:KeyName}" \
  --output table
```

Confirm Security Group association:
```bash
aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query "Reservations[].Instances[].SecurityGroups[].{GroupId:GroupId,Name:GroupName}" \
  --output table
```

Confirm Key Pair exists:
```bash
aws ec2 describe-key-pairs \
  --key-names nautilus-kp \
  --query "KeyPairs[].{Name:KeyName,Type:KeyType}" \
  --output table
```

---

## 🔍 Complete Verification

Verify all key parameters with a single query:
```bash
aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query "Reservations[].Instances[].{Name:Tags[?Key=='Name']|[0].Value,InstanceId:InstanceId,Type:InstanceType,State:State.Name,KeyPair:KeyName,SecurityGroups:SecurityGroups[].GroupName}" \
  --output table
```

---

## ✅ Final Result

The required EC2 instance was successfully launched and verified.

| Configuration | Result |
| :--- | :--- |
| **Instance Name** | `nautilus-ec2` |
| **AMI** | Amazon Linux |
| **Instance Type** | `t2.micro` |
| **Key Pair** | `nautilus-kp` |
| **Key Type** | RSA |
| **Security Group** | Default |
| **Region** | `us-east-1` |
| **State** | `running` |
| **Method** | AWS CLI |

---

## 🔐 Security & Cost Considerations

* **Key Protection:** Ensure key files are never committed to version control:
  ```bash
  echo "*.pem" >> .gitignore
  ```
* **Security Groups:** The default security group is used for lab simplicity, but production instances should always be deployed with dedicated security groups granting least-privilege network access.
* **Resource Cleanup:** After lab evaluation, instances can be terminated to avoid incurring ongoing charges:
  ```bash
  aws ec2 terminate-instances --instance-ids "$INSTANCE_ID"
  aws ec2 wait instance-terminated --instance-ids "$INSTANCE_ID"
  ```

---

## 🧠 What I Learned & Key Takeaway

Launching an EC2 instance brings multiple foundational AWS components together into a unified workflow:

```text
                  EC2 Instance (nautilus-ec2)
                               │
       ┌───────────────────────┼───────────────────────┐
       ↓                       ↓                       ↓
   AMI (OS)             Key Pair (SSH)      Security Group (Firewall)
```

Using the AWS CLI allows discovering prerequisites, launching compute instances, and verifying provisioning in a repeatable, scriptable manner suitable for real-world automation pipelines.

> *This was Task 6 of 50 AWS tasks in the AWS phase of my KodeKloud 100 Days of Cloud Challenge.*
