# AWS Task 22 — Configuring Secure SSH Access to an EC2 Instance

## 📌 Task Overview

In this task, the Nautilus DevOps team required a dedicated EC2 instance named `datacenter-ec2` configured for passwordless, key-based root SSH access directly from the `aws-client` landing host. The objective was to generate an RSA SSH key pair on the landing host, launch an Ubuntu instance in `us-east-1`, bootstrap initial SSH access via a temporary EC2 Key Pair, deploy the `aws-client` public key to `/root/.ssh/authorized_keys`, and enforce `PermitRootLogin prohibit-password` in SSHD.

| Requirement | Value |
| :--- | :--- |
| **EC2 Instance Name** | `datacenter-ec2` |
| **Instance Type** | `t2.micro` |
| **AWS Region Context** | `us-east-1` |
| **Client Key Location** | `/root/.ssh/id_rsa` and `/root/.ssh/id_rsa.pub` on `aws-client` |
| **Target Identity** | `root` user passwordless SSH on `datacenter-ec2` |
| **Method** | AWS CLI, OpenSSH Tooling, & Linux Administration |

---

## 🎯 Objective

Establish passwordless, key-authenticated SSH access from a client host to an EC2 target:

```text
Generate SSH Key on aws-client ──► Launch EC2 with Bootstrap Key ──► Copy Public Key to Target /root/.ssh/authorized_keys ──► Configure SSHD ──► Test Direct Root Access
```

---

## 🛠️ Prerequisites & Key Generation

Ensure active AWS credentials and generate the local client RSA key pair on the `aws-client` host if missing:

```bash
export AWS_DEFAULT_REGION=us-east-1

# Generate 4096-bit RSA key pair without a passphrase if it does not exist
if [ ! -f /root/.ssh/id_rsa ]; then
  mkdir -p /root/.ssh
  ssh-keygen -t rsa -b 4096 -f /root/.ssh/id_rsa -N ""
fi

chmod 600 /root/.ssh/id_rsa
chmod 644 /root/.ssh/id_rsa.pub
```

---

## 1. Create Temporary Bootstrap EC2 Key Pair

Create a temporary EC2 Key Pair to enable initial connectivity as the default `ubuntu` user:

```bash
aws ec2 create-key-pair \
  --key-name datacenter-bootstrap-key \
  --query "KeyMaterial" \
  --output text > /root/.ssh/datacenter-bootstrap-key.pem

chmod 400 /root/.ssh/datacenter-bootstrap-key.pem
```

---

## 2. Resolve Environment & Launch EC2 Instance

Query the target AMI, default VPC, subnet, and default security group, then launch `datacenter-ec2`:

```bash
# Resolve networking and image parameters
AMI_ID=$(aws ec2 describe-images --owners 099720109477 --filters "Name=name,Values=ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*" "Name=state,Values=available" --query "Images | sort_by(@, &CreationDate)[-1].ImageId" --output text)
VPC_ID=$(aws ec2 describe-vpcs --filters "Name=is-default,Values=true" --query "Vpcs[0].VpcId" --output text)
SUBNET_ID=$(aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID" --query "Subnets[0].SubnetId" --output text)
SG_ID=$(aws ec2 describe-security-groups --filters "Name=vpc-id,Values=$VPC_ID" "Name=group-name,Values=default" --query "SecurityGroups[0].GroupId" --output text)

# Launch Instance
INSTANCE_ID=$(aws ec2 run-instances \
  --image-id "$AMI_ID" \
  --instance-type t2.micro \
  --subnet-id "$SUBNET_ID" \
  --security-group-ids "$SG_ID" \
  --key-name datacenter-bootstrap-key \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=datacenter-ec2}]' \
  --query "Instances[0].InstanceId" \
  --output text)

aws ec2 wait instance-status-ok --instance-ids "$INSTANCE_ID"

PUBLIC_IP=$(aws ec2 describe-instances --instance-ids "$INSTANCE_ID" --query "Reservations[0].Instances[0].PublicIpAddress" --output text)
echo "Target Instance Public IP: $PUBLIC_IP"
```

---

## 3. Authorize Client Public Key for Target Root User

Copy the `aws-client` public key (`/root/.ssh/id_rsa.pub`) into the EC2 instance's `/root/.ssh/authorized_keys` file via remote SSH execution:

```bash
CLIENT_PUB_KEY=$(cat /root/.ssh/id_rsa.pub)

# Inject public key and configure SSH permissions on the remote target
ssh -i /root/.ssh/datacenter-bootstrap-key.pem -o StrictHostKeyChecking=no ubuntu@$PUBLIC_IP << EOF
sudo su -
mkdir -p /root/.ssh
echo "$CLIENT_PUB_KEY" >> /root/.ssh/authorized_keys
chmod 700 /root/.ssh
chmod 600 /root/.ssh/authorized_keys
chown -R root:root /root/.ssh

# Enable key-based root login in OpenSSH
sed -i 's/^#\?PermitRootLogin.*/PermitRootLogin prohibit-password/' /etc/ssh/sshd_config
sshd -t && systemctl restart ssh
EOF
```

---

## 🔍 Complete Verification

Test passwordless SSH root execution directly from `aws-client` using `/root/.ssh/id_rsa`:

```bash
ssh -i /root/.ssh/id_rsa -o StrictHostKeyChecking=no root@$PUBLIC_IP 'whoami && hostname'
```

**Expected output:**
```text
root
ip-172-31-x-x
```

Verify instance details via AWS CLI:

```bash
aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query "Reservations[].Instances[].{Name:Tags[?Key=='Name']|[0].Value,InstanceId:InstanceId,State:State.Name,PublicIP:PublicIpAddress}" \
[O  --output table
```

---

## 🧠 Key Concepts & Takeaways

| Concept | Description |
| :--- | :--- |
| **Asymmetric Key Exchange** | The private key (`id_rsa`) remains strictly on `aws-client` with `0600` permissions. The public key (`id_rsa.pub`) is copied to the remote server's `authorized_keys` file. |
| **SSH File Permissions** | OpenSSH strictly enforces permission checks (`StrictModes`). The `/root/.ssh` directory requires `0700` and `authorized_keys` requires `0600` owned by `root:root`. Loose permissions cause `Permission denied (publickey)` errors. |
| **PermitRootLogin Modes** | Setting `PermitRootLogin prohibit-password` (or `without-password`) allows secure key-based authentication for the `root` account while blocking insecure password attempts. |
| **Bootstrap Key Workaround** | EC2 instances initialized without Cloud-Init SSH key injection for `root` require a temporary default user session (`ubuntu`) with `sudo` escalation to populate root's key directory. |

> *This was Task 22 of 50 AWS tasks in the AWS phase of my KodeKloud 100 Days of Cloud Challenge.*
