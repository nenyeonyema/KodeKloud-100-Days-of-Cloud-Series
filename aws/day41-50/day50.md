# Task 50 — EBS Volume Expansion for EC2 Instance

## Overview

This task expands the EBS root volume of an existing EC2 instance from 8 GiB
to 12 GiB and ensures the expanded storage is immediately available inside
the instance without downtime or disruption. The process involves modifying
the EBS volume via AWS CLI, then extending the partition and filesystem
from within the instance to reflect the new size.

---

## Architecture

```
AWS Console / CLI
      ↓
EBS Volume Modification
(8 GiB → 12 GiB)
      ↓
EC2 Instance (datacenter-ec2)
      ↓
┌─────────────────────────────────┐
│  Block Device: /dev/xvda (12G)  │
│  ├─ xvda1    (partition — 8G)   │  ← before growpart
│  ├─ xvda127  (1M)               │
│  └─ xvda128  (10M /boot/efi)    │
└─────────────────────────────────┘
      ↓ sudo growpart /dev/xvda 1
      ↓ sudo xfs_growfs /
┌─────────────────────────────────┐
│  Block Device: /dev/xvda (12G)  │
│  ├─ xvda1    (partition — 12G)  │  ← after growpart + xfs_growfs
│  ├─ xvda127  (1M)               │
│  └─ xvda128  (10M /boot/efi)    │
└─────────────────────────────────┘
```

---

## Resources

| Resource | Name | Description |
|---|---|---|
| EC2 Instance | `datacenter-ec2` | Instance with storage to expand |
| EBS Volume | Auto-identified | Root volume attached to the instance |
| SSH Key | `datacenter-keypair.pem` | At `/root/` on aws-client |

---

## Prerequisites

- AWS CLI configured with appropriate credentials
- Region: `us-east-1`
- SSH key at `/root/datacenter-keypair.pem` on aws-client
- EC2 instance `datacenter-ec2` running with an 8 GiB root volume

---

## Step-by-Step Guide

### Step 1 — Identify the instance and volume

```bash
# Get instance ID
EC2_ID=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=datacenter-ec2" \
            "Name=instance-state-name,Values=running" \
  --query 'Reservations[0].Instances[0].InstanceId' \
  --output text)
echo "EC2 ID: $EC2_ID"
```

```bash
# Get public IP
EC2_IP=$(aws ec2 describe-instances \
  --instance-ids $EC2_ID \
  --query 'Reservations[0].Instances[0].PublicIpAddress' \
  --output text)
echo "EC2 IP: $EC2_IP"
```

```bash
# Get availability zone — always fetch dynamically
EC2_AZ=$(aws ec2 describe-instances \
  --instance-ids $EC2_ID \
  --query 'Reservations[0].Instances[0].Placement.AvailabilityZone' \
  --output text)
echo "EC2 AZ: $EC2_AZ"
```

```bash
# Get the attached root volume ID
VOL_ID=$(aws ec2 describe-instances \
  --instance-ids $EC2_ID \
  --query 'Reservations[0].Instances[0].BlockDeviceMappings[0].Ebs.VolumeId' \
  --output text)
echo "Volume ID: $VOL_ID"
```

```bash
# Confirm current volume size (should show 8 GiB)
aws ec2 describe-volumes \
  --volume-ids $VOL_ID \
  --query 'Volumes[0].{ID:VolumeId,Size:Size,State:State,Type:VolumeType}' \
  --output table
```

---

### Step 2 — Expand the EBS volume

```bash
# Modify volume from 8 GiB to 12 GiB
aws ec2 modify-volume \
  --volume-id $VOL_ID \
  --size 12 \
  --region us-east-1
```

```bash
# Check modification state — wait until it shows "completed"
aws ec2 describe-volumes-modifications \
  --volume-ids $VOL_ID \
  --query 'VolumesModifications[0].{State:ModificationState,OldSize:OriginalSize,NewSize:TargetSize}' \
  --output table
```

Expected output when complete:
```
-----------------------------------------
|      DescribeVolumesModifications     |
+----------+----------+-----------------+
|  NewSize |  OldSize |  State          |
+----------+----------+-----------------+
|  12      |  8       |  completed      |
+----------+----------+-----------------+
```

Re-run if state shows `modifying` — takes ~1 minute:
```bash
aws ec2 describe-volumes-modifications \
  --volume-ids $VOL_ID \
  --query 'VolumesModifications[0].ModificationState' \
  --output text
```

---

### Step 3 — SSH into the instance

```bash
chmod 400 /root/datacenter-keypair.pem
```

Try direct SSH first:
```bash
ssh -i /root/datacenter-keypair.pem \
  -o StrictHostKeyChecking=no \
  ec2-user@$EC2_IP
```

If `Permission denied`, use EC2 Instance Connect API:
```bash
PUB_KEY=$(ssh-keygen -y -f /root/datacenter-keypair.pem)

aws ec2-instance-connect send-ssh-public-key \
  --instance-id $EC2_ID \
  --instance-os-user ec2-user \
  --ssh-public-key "$PUB_KEY" \
  --availability-zone $EC2_AZ \
  --region us-east-1 && \
ssh -i /root/datacenter-keypair.pem \
  -o StrictHostKeyChecking=no \
  ec2-user@$EC2_IP
```

> **Note:** Amazon Linux instances use `ec2-user`. Ubuntu instances use
> `ubuntu`. Always check the AMI type before connecting.

---

### Step 4 — Check current disk layout

Once inside the instance:

```bash
lsblk
```

Expected output before expanding partition:
```
NAME      MAJ:MIN RM SIZE RO TYPE MOUNTPOINTS
xvda      202:0    0  12G  0 disk
├─xvda1   202:1    0   8G  0 part /
├─xvda127 259:0    0   1M  0 part
└─xvda128 259:1    0  10M  0 part /boot/efi
```

The disk (`xvda`) shows 12G but the partition (`xvda1`) still shows 8G.
The filesystem must be extended to use the new space.

---

### Step 5 — Grow the partition

```bash
sudo growpart /dev/xvda 1
```

Expected output:
```
CHANGED: partition=1 start=4096 old: size=16773087 end=16777183
         new: size=25161695 end=25165791
```

---

### Step 6 — Resize the filesystem

**For XFS filesystem (Amazon Linux — most common):**
```bash
sudo xfs_growfs /
```

**For ext4 filesystem (Ubuntu):**
```bash
sudo resize2fs /dev/xvda1
```

> To check which filesystem type is in use:
> ```bash
> df -T /
> ```

---

### Step 7 — Verify the expanded size

```bash
lsblk
```

Expected output after expansion:
```
NAME      MAJ:MIN RM SIZE RO TYPE MOUNTPOINTS
xvda      202:0    0  12G  0 disk
├─xvda1   202:1    0  12G  0 part /          ← now 12G
├─xvda127 259:0    0   1M  0 part
└─xvda128 259:1    0  10M  0 part /boot/efi
```

```bash
df -h /
```

Expected output:
```
Filesystem      Size  Used Avail Use% Mounted on
/dev/xvda1       12G  2.1G  9.9G  18% /
```

Root partition now reflects 12 GiB — task complete.

```bash
exit
```

---

## Key Concepts

### Why two steps are needed (modify + growpart + resize)

Expanding an EBS volume via AWS only increases the raw disk size. The
partition table and filesystem inside the OS are not automatically updated.
Three separate operations are required:

| Step | Tool | What it does |
|---|---|---|
| 1 | `aws ec2 modify-volume` | Expands the physical EBS disk |
| 2 | `sudo growpart` | Extends the partition to use new disk space |
| 3 | `sudo xfs_growfs` or `sudo resize2fs` | Extends the filesystem to fill the partition |

### Zero downtime expansion
EBS volumes can be expanded while the instance is running. No reboot or
stop is required. The `growpart` and filesystem resize commands are also
online operations — the instance stays fully available throughout.

### growpart partition numbering
The number `1` in `sudo growpart /dev/xvda 1` refers to the partition
number, not a flag. It tells growpart to expand partition 1 on disk xvda.
Always check `lsblk` first to confirm the partition number.

### XFS vs ext4
Amazon Linux uses XFS filesystem by default — use `xfs_growfs /`.
Ubuntu uses ext4 by default — use `resize2fs /dev/xvda1`.
Running the wrong command will produce an error but cause no harm.

### EC2 Instance Connect API
When the local PEM key doesn't match the instance (common in lab
environments), `send-ssh-public-key` pushes a temporary public key that
grants access for 60 seconds. SSH must be executed immediately after within
that window.

```bash
# Always fetch AZ dynamically — never hardcode us-east-1a
EC2_AZ=$(aws ec2 describe-instances \
  --instance-ids $EC2_ID \
  --query 'Reservations[0].Instances[0].Placement.AvailabilityZone' \
  --output text)
```

---

## Common Issues and Fixes

### Permission denied (publickey)
**Cause:** PEM key does not match the instance key pair.
**Fix:** Use EC2 Instance Connect API to push a temporary key.

```bash
PUB_KEY=$(ssh-keygen -y -f /root/datacenter-keypair.pem)
aws ec2-instance-connect send-ssh-public-key \
  --instance-id $EC2_ID \
  --instance-os-user ec2-user \
  --ssh-public-key "$PUB_KEY" \
  --availability-zone $EC2_AZ \
  --region us-east-1
```

### EC2InstanceNotFoundException on Instance Connect
**Cause:** Wrong availability zone — hardcoded AZ doesn't match instance.
**Fix:** Always fetch the AZ dynamically using `describe-instances`.

### Partition still shows old size after modify-volume
**Cause:** AWS volume modification is complete but partition not yet grown.
**Fix:** Run `sudo growpart /dev/xvda 1` inside the instance.

### Filesystem still shows old size after growpart
**Cause:** Partition grown but filesystem not resized.
**Fix:** Run `sudo xfs_growfs /` (XFS) or `sudo resize2fs /dev/xvda1` (ext4).

### SSH connection timeout
**Cause:** Security group does not allow port 22 from aws-client IP.
**Fix:**
```bash
MY_IP=$(curl -s ifconfig.me)
SG_ID=$(aws ec2 describe-instances \
  --instance-ids $EC2_ID \
  --query 'Reservations[0].Instances[0].SecurityGroups[0].GroupId' \
  --output text)

aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID \
  --protocol tcp --port 22 \
  --cidr ${MY_IP}/32
```

---

## Verification Checklist

- [ ] Volume ID identified from instance block device mappings
- [ ] `modify-volume` completed with state `completed`
- [ ] Successfully SSH'd into instance
- [ ] `lsblk` shows disk at 12G before growpart
- [ ] `growpart` output shows `CHANGED`
- [ ] `xfs_growfs` or `resize2fs` completed without errors
- [ ] `lsblk` shows `xvda1` partition at 12G
- [ ] `df -h /` shows root filesystem at ~12G

---

## Quick Reference Commands

```bash
# Check volume modification status
aws ec2 describe-volumes-modifications \
  --volume-ids $VOL_ID \
  --query 'VolumesModifications[0].ModificationState' \
  --output text

# Check disk layout inside instance
lsblk

# Check filesystem type
df -T /

# Check disk usage
df -h /

# Grow partition (partition number 1)
sudo growpart /dev/xvda 1

# Grow XFS filesystem
sudo xfs_growfs /

# Grow ext4 filesystem
sudo resize2fs /dev/xvda1
```

---

## Author
Nenye — Cloud & DevOps Engineer
Stack: AWS EC2 · EBS · IAM · Instance Connect
Region: us-east-1
