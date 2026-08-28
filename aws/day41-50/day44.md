# AWS Task 44 — Auto Scaling & Load Balancing: Deploying a Highly Available Nginx Web Application on AWS

## Overview

This project documents the setup of a highly available web application infrastructure on AWS using an Auto Scaling Group (ASG) and an Application Load Balancer (ALB). The team needed EC2 instances running Nginx to serve web traffic, automatically scale based on CPU utilization, and distribute incoming requests evenly across healthy instances. A launch template was created with a UserData script to install and start Nginx at boot, and a CPU-based target tracking scaling policy was configured to maintain performance under load.

---

## Architecture

```
                         Internet
                             │
                             ▼ port 80
                 ┌───────────────────────┐
                 │  Application Load     │
                 │  Balancer (xfusion-alb│
                 │  internet-facing)     │
                 └───────────┬───────────┘
                             │
                             ▼ HTTP:80
                 ┌───────────────────────┐
                 │    Target Group       │
                 │    (xfusion-tg)       │
                 │  Health: GET /        │
                 └───────────┬───────────┘
                             │
              ┌──────────────┴──────────────┐
              │                             │
              ▼                             ▼
   ┌─────────────────┐           ┌─────────────────┐
   │   EC2 Instance  │           │   EC2 Instance  │
   │  (xfusion-web)  │           │  (xfusion-web)  │
   │  Amazon Linux 2 │           │  Amazon Linux 2 │
   │  t2.micro       │           │  t2.micro       │
   │  Nginx running  │           │  Nginx running  │
   └─────────────────┘           └─────────────────┘
              │                             │
              └──────────────┬──────────────┘
                             │
                 ┌───────────────────────┐
                 │  Auto Scaling Group   │
                 │  (xfusion-asg)        │
                 │  Min: 1  Max: 2       │
                 │  Desired: 1           │
                 │  CPU Target: 50%      │
                 └───────────────────────┘
                             │
                 ┌───────────────────────┐
                 │   Launch Template     │
                 │ (xfusion-launch-      │
                 │  template)            │
                 │  AMI: Amazon Linux 2  │
                 │  Type: t2.micro       │
                 │  SG: port 80 open     │
                 │  UserData: Nginx      │
                 └───────────────────────┘
```

---

## Prerequisites

- AWS CLI configured with valid credentials
- IAM permissions for:
  - `ec2:CreateLaunchTemplate`
  - `ec2:CreateSecurityGroup`
  - `ec2:AuthorizeSecurityGroupIngress`
  - `elasticloadbalancing:CreateLoadBalancer`
  - `elasticloadbalancing:CreateTargetGroup`
  - `elasticloadbalancing:CreateListener`
  - `autoscaling:CreateAutoScalingGroup`
  - `autoscaling:PutScalingPolicy`
- Region: `us-east-1`

---

## What Was Built

| Resource | Name | Configuration |
|---|---|---|
| Security Group | `xfusion-sg` | Inbound TCP port 80 from `0.0.0.0/0` |
| Launch Template | `xfusion-launch-template` | Amazon Linux 2, t2.micro, Nginx via UserData |
| Target Group | `xfusion-tg` | HTTP:80, health check on `/` |
| Load Balancer | `xfusion-alb` | Internet-facing, ALB, port 80 listener |
| Auto Scaling Group | `xfusion-asg` | Min:1, Desired:1, Max:2, ELB health checks |
| Scaling Policy | CPU Target Tracking | 50% CPU utilization threshold |

---

## UserData Script (Nginx Installation)

```bash
#!/bin/bash
exec > /var/log/userdata.log 2>&1
yum update -y
amazon-linux-extras install nginx1 -y
systemctl start nginx
systemctl enable nginx
systemctl status nginx
```

> **Important:** On Amazon Linux 2, Nginx must be installed via
> `amazon-linux-extras install nginx1 -y` — not `yum install nginx`.
> Using `yum install nginx` directly will silently fail and Nginx
> will never start, causing all health checks to fail with HTTP 502.

---

## Step-by-Step Implementation

### 1. Verify Credentials and Get Default VPC

```bash
aws sts get-caller-identity

VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=isDefault,Values=true" \
  --query 'Vpcs[0].VpcId' \
  --output text --region us-east-1)
echo "VPC ID: $VPC_ID"

SUBNET_IDS=$(aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'Subnets[*].SubnetId' \
  --output text --region us-east-1 | tr '\t' ',')
echo "Subnet IDs: $SUBNET_IDS"
```

### 2. Create Security Group

```bash
SG_ID=$(aws ec2 create-security-group \
  --group-name xfusion-sg \
  --description "Security group for xfusion web servers - HTTP port 80" \
  --vpc-id $VPC_ID \
  --query 'GroupId' \
  --output text --region us-east-1)
echo "Security Group ID: $SG_ID"

aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0 \
  --region us-east-1

echo "✅ Security group configured"
```

### 3. Get Latest Amazon Linux 2 AMI

```bash
AMI_ID=$(aws ec2 describe-images \
  --owners amazon \
  --filters \
    "Name=name,Values=amzn2-ami-hvm-*-x86_64-gp2" \
    "Name=state,Values=available" \
  --query 'sort_by(Images, &CreationDate)[-1].ImageId' \
  --output text --region us-east-1)
echo "AMI ID: $AMI_ID"
```

### 4. Prepare UserData Script

```bash
[Ocat > /tmp/userdata.sh << 'EOF'
#!/bin/bash
exec > /var/log/userdata.log 2>&1
yum update -y
amazon-linux-extras install nginx1 -y
systemctl start nginx
systemctl enable nginx
EOF

USERDATA=$(base64 -w 0 /tmp/userdata.sh)
echo "✅ UserData encoded"
```

### 5. Create Launch Template

```bash
aws ec2 create-launch-template \
  --launch-template-name xfusion-launch-template \
  --version-description "v1 - Nginx on Amazon Linux 2" \
  --launch-template-data "{
    \"ImageId\": \"$AMI_ID\",
    \"InstanceType\": \"t2.micro\",
    \"NetworkInterfaces\": [
      {
        \"AssociatePublicIpAddress\": true,
        \"DeviceIndex\": 0,
        \"Groups\": [\"$SG_ID\"]
      }
    ],
    \"UserData\": \"$USERDATA\",
    \"TagSpecifications\": [
      {
        \"ResourceType\": \"instance\",
        \"Tags\": [{\"Key\": \"Name\", \"Value\": \"xfusion-web\"}]
      }
    ]
  }" \
  --region us-east-1

LT_ID=$(aws ec2 describe-launch-templates \
  --launch-template-names xfusion-launch-template \
  --query 'LaunchTemplates[0].LaunchTemplateId' \
  --output text --region us-east-1)
echo "Launch Template ID: $LT_ID"
```

### 6. Create Target Group

```bash
TG_ARN=$(aws elbv2 create-target-group \
  --name xfusion-tg \
  --protocol HTTP \
  --port 80 \
  --vpc-id $VPC_ID \
  --target-type instance \
  --health-check-protocol HTTP \
  --health-check-path "/" \
  --health-check-interval-seconds 30 \
  --health-check-timeout-seconds 5 \
  --healthy-threshold-count 2 \
  --unhealthy-threshold-count 2 \
  --query 'TargetGroups[0].TargetGroupArn' \
  --output text --region us-east-1)
echo "Target Group ARN: $TG_ARN"
```

### 7. Create Application Load Balancer

```bash
ALB_ARN=$(aws elbv2 create-load-balancer \
  --name xfusion-alb \
  --subnets $(echo $SUBNET_IDS | tr ',' ' ') \
  --security-groups $SG_ID \
  --scheme internet-facing \
  --type application \
  --ip-address-type ipv4 \
  --query 'LoadBalancers[0].LoadBalancerArn' \
  --output text --region us-east-1)
echo "ALB ARN: $ALB_ARN"

ALB_DNS=$(aws elbv2 describe-load-balancers \
  --load-balancer-arns $ALB_ARN \
  --query 'LoadBalancers[0].DNSName' \
  --output text --region us-east-1)
echo "ALB DNS: $ALB_DNS"
```

### 8. Create ALB Listener on Port 80

```bash
aws elbv2 create-listener \
  --load-balancer-arn $ALB_ARN \
  --protocol HTTP \
  --port 80 \
  --default-actions Type=forward,TargetGroupArn=$TG_ARN \
  --region us-east-1

echo "✅ ALB listener created on port 80"
```

### 9. Create Auto Scaling Group

```bash
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name xfusion-asg \
  --launch-template "LaunchTemplateId=$LT_ID,Version=1" \
  --min-size 1 \
  --max-size 2 \
  --desired-capacity 1 \
  --target-group-arns $TG_ARN \
  --vpc-zone-identifier "$SUBNET_IDS" \
  --health-check-type ELB \
  --health-check-grace-period 120 \
  --region us-east-1

echo "✅ Auto Scaling Group created"
```

### 10. Create CPU Target Tracking Scaling Policy

```bash
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name xfusion-asg \
  --policy-name xfusion-cpu-scaling-policy \
  --policy-type TargetTrackingScaling \
  --target-tracking-configuration "{
    \"PredefinedMetricSpecification\": {
      \"PredefinedMetricType\": \"ASGAverageCPUUtilization\"
    },
    \"TargetValue\": 50.0
  }" \
  --region us-east-1

echo "✅ CPU scaling policy set to 50% target"
```

### 11. Wait and Test Nginx via ALB

```bash
echo "⏳ Waiting 2 minutes for instance to launch and Nginx to start..."
sleep 120

echo "=== Target Health ==="
[Iaws elbv2 describe-target-health \
  --target-group-arn $TG_ARN \
  --region us-east-1

echo "=== Testing ALB ==="
curl -s -o /dev/null -w "HTTP Status: %{http_code}\n" http://$ALB_DNS
curl -s http://$ALB_DNS | grep -i "<title>"
```

---

## Verification

**Expected output:**
```
HTTP Status: 200
<title>Welcome to nginx!</title>  ✅
```

**Confirm ASG and ALB configuration:**
```bash
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names xfusion-asg \
  --query 'AutoScalingGroups[0].{Min:MinSize,Max:MaxSize,Desired:DesiredCapacity,Health:HealthCheckType}' \
  --output table --region us-east-1

aws elbv2 describe-load-balancers \
  --names xfusion-alb \
  --query 'LoadBalancers[0].{Name:LoadBalancerName,DNS:DNSName,State:State.Code}' \
  --output table --region us-east-1
```

---

## Console Verification Steps

For those who prefer the AWS Console:

1. **EC2 → Launch Templates** — confirm `xfusion-launch-template` exists
2. **EC2 → Auto Scaling Groups** — confirm `xfusion-asg` with Min:1, Max:2, Desired:1
3. **EC2 → Auto Scaling Groups → Automatic Scaling tab** — confirm CPU target tracking policy at 50%
4. **EC2 → Load Balancers** — confirm `xfusion-alb` is `Active` and internet-facing
5. **EC2 → Target Groups → xfusion-tg → Targets tab** — confirm instance is `healthy`
6. **Browser** — visit `http://<ALB-DNS>` and confirm Nginx welcome page loads

---

## Troubleshooting Checklist

| Check | Command | Expected Result |
|---|---|---|
| Instance running | `describe-auto-scaling-groups` | `LifecycleState: InService` |
| Instance has public IP | `describe-instances` | `PublicIpAddress` not None |
| Nginx responding directly | `curl http://<instance-ip>` | `HTTP Status: 200` |
| Target health | `describe-target-health` | `State: healthy` |
| ALB responding | `curl http://<ALB-DNS>` | `HTTP Status: 200` |
| Security group rules | `describe-security-groups` | Inbound TCP 80 from `0.0.0.0/0` |
[O
---

## Common Mistakes

| Mistake | Consequence |
|---|---|
| Using `yum install nginx` on Amazon Linux 2 | Nginx not found — UserData completes but Nginx never installs |
| Not setting `AssociatePublicIpAddress: true` in NetworkInterfaces | Instance launches with no public IP — direct instance test impossible |
| Using `SecurityGroupIds` with `NetworkInterfaces` in same template | AWS rejects the request — security groups must be inside `NetworkInterfaces` block |
| Not waiting for health check grace period | Target shows unhealthy before Nginx finishes starting — premature failure |
| ALB and instance in different security groups | ALB cannot reach instance on port 80 — HTTP 502 returned |
| Subnet IDs tab-separated not comma-separated | ASG vpc-zone-identifier rejects the value |

---

## Key Lesson

On **Amazon Linux 2**, Nginx is not available in the default `yum` repositories. It must be installed through the Amazon Linux Extras library:

```bash
amazon-linux-extras install nginx1 -y
```

Using `yum install nginx` will silently produce no error but install nothing — leaving the instance running with no web server, causing the ALB health checks to fail and returning HTTP 502 to all clients. Always verify Nginx is running directly on the instance IP before debugging the ALB.

Always follow this order when setting up ASG + ALB:

```
Security Group → AMI lookup → Launch Template → Target Group
→ ALB → Listener → ASG → Scaling Policy → Verify health → Test DNS
```

Never test the ALB before confirming the target is `healthy` in the target group — an unhealthy target always returns 502 regardless of ALB configuration.

---

## Technologies Used

- **AWS EC2 Launch Template** — Instance configuration with UserData automation
- **AWS Auto Scaling Group** — Automatic instance management with min/max/desired capacity
- **AWS Application Load Balancer** — Internet-facing HTTP traffic distribution
- **AWS Target Group** — Health-checked instance pool for the ALB
- **Target Tracking Scaling Policy** — CPU-based automatic scale-out at 50% threshold
- **Amazon Linux 2** — Base OS with `amazon-linux-extras` for Nginx
- **Nginx** — Web server serving the default welcome page
- **AWS CLI** — Full infrastructure provisioning via command line
