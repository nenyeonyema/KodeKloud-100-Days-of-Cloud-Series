# AWS Task 25 — EC2 Instance Provisioning and CloudWatch Alarm Configuration

## 📌 Task Overview

The objective of this task was to launch an Ubuntu EC2 instance (`datacenter-ec2`) and configure an Amazon CloudWatch alarm (`datacenter-alarm`) to monitor its CPU utilization. The alarm was set to trigger and publish a notification to an existing Simple Notification Service (SNS) topic (`datacenter-sns-topic`) whenever average CPU utilization reaches or exceeds 90% for a consecutive 5-minute evaluation period in `us-east-1`.

| Requirement | Value |
| :--- | :--- |
| **EC2 Name / AMI** | `datacenter-ec2` (Ubuntu 24.04 LTS / `t2.micro`) |
| **Alarm Name** | `datacenter-alarm` |
| **Metric Monitored** | `CPUUtilization` (Namespace: `AWS/EC2`) |
| **Threshold / Statistic** | `>= 90%` | `Average` |
| **Evaluation Period** | 1 period of 300 seconds (5 minutes) |
| **Notification Action** | `datacenter-sns-topic` |
| **AWS Region Context** | `us-east-1` |

---

## 🎯 Architecture & Monitoring Flow

```text
                  ┌──────────────────────┐
                  │    EC2 Instance      │
                  │   datacenter-ec2     │
                  └──────────┬───────────┘
                             │
                             │ CPUUtilization
                             ▼
                  ┌──────────────────────┐
                  │      CloudWatch      │
                  │   datacenter-alarm   │
                  └──────────┬───────────┘
                             │
                  CPU >= 90% for 5 minutes
                             │
                             ▼
                  ┌──────────────────────┐
                  │       SNS Topic      │
                  │ datacenter-sns-topic │
                  └──────────────────────┘
🛠️ Implementation & Verification Commands
Bash
# ==========================================
# 1. CONFIGURE REGION & RETRIEVE AMIS/NETWORKING
# ==========================================
aws configure set region us-east-1

# Discover latest Ubuntu 24.04 LTS AMI ID
AMI_ID=$(aws ec2 describe-images \
  --owners 099720109477 \
  --filters \
    "Name=name,Values=ubuntu/images/hvm-ssd-gp3/ubuntu-noble-24.04-amd64-server-*" \
    "Name=state,Values=available" \
    "Name=architecture,Values=x86_64" \
  --query "sort_by(Images,&CreationDate)[-1].ImageId" \
  --output text)

# Discover default Subnet and Security Group IDs
SUBNET_ID=$(aws ec2 describe-subnets --query "Subnets[0].SubnetId" --output text)
SG_ID=$(aws ec2 describe-security-groups --query "SecurityGroups[0].GroupId" --output text)

echo "AMI: $AMI_ID | Subnet: $SUBNET_ID \vert{} SG:$SG_ID"

# ==========================================
# 2. LAUNCH & TAG EC2 INSTANCE
# ==========================================
INSTANCE_ID=$(aws ec2 run-instances \
  --image-id "$AMI_ID" \
  --instance-type t2.micro \
  --subnet-id "$SUBNET_ID" \
  --security-group-ids "$SG_ID" \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=datacenter-ec2}]' \
  --query "Instances[0].InstanceId" \
  --output text)

# Wait until instance is in running state
aws ec2 wait instance-running --instance-ids "$INSTANCE_ID"

# ==========================================
# 3. DISCOVER SNS TOPIC ARN
# ==========================================
SNS_TOPIC_ARN=$(aws sns list-topics \
  --query "Topics[?contains(TopicArn, 'datacenter-sns-topic')].TopicArn | [0]" \
  --output text)

echo "Target SNS Topic ARN: $SNS_TOPIC_ARN"

# ==========================================
# 4. CREATE CLOUDWATCH ALARM
# ==========================================
aws cloudwatch put-metric-alarm \
  --alarm-name datacenter-alarm \
  --alarm-description "Alarm when EC2 CPU utilization reaches or exceeds 90 percent" \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 300 \
  --evaluation-periods 1 \
  --threshold 90 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --dimensions Name=InstanceId,Value="$INSTANCE_ID" \
  --alarm-actions "$SNS_TOPIC_ARN"

# ==========================================
# 5. VERIFICATION & INTEGRITY CHECKS
# ==========================================
# Verify EC2 Instance details
aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query "Reservations[0].Instances[0].{InstanceId:InstanceId,State:State.Name,PrivateIP:PrivateIpAddress,PublicIP:PublicIpAddress}" \
  --output table

# Verify CloudWatch Alarm parameters
aws cloudwatch describe-alarms \
  --alarm-names datacenter-alarm \
  --query "MetricAlarms[0].{Name:AlarmName, State:StateValue, Metric:MetricName, Statistic:Statistic, Period:Period, EvaluationPeriods:EvaluationPeriods, Threshold:Threshold}" \
  --output table
Note: A query returning an OK or INSUFFICIENT_DATA state (initial metric collection phase) along with matching parameters (CPUUtilization, Average, 300s period, threshold 90.0) confirms successful alarm attachment to the target EC2 instance.
