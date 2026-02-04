# Task 13
> The Nautilus DevOps team is strategizing the migration of a portion of their infrastructure to the AWS cloud. Recognizing the scale of this undertaking, they have opted to approach the migration in incremental steps rather than as a single massive transition. 
> To achieve this, they have segmented large tasks into smaller, more manageable units. This granular approach enables the team to execute the migration in gradual phases, ensuring smoother implementation and minimizing disruption to ongoing operations. 
> By breaking down the migration into smaller tasks, the Nautilus DevOps team can systematically progress through each stage, allowing for better control, risk mitigation, and optimization of resources throughout the migration process.
>
> For this task, create an AMI from an existing EC2 instance named devops-ec2 with the following requirement:
>
> Name of the AMI should be devops-ec2-ami, make sure AMI is in available state.


### Create an AMI from an existing EC2 Instance named devops-ec2


1. Get the Instance ID for devops-ec2

```
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=devops-ec2" \
  --query "Reservations[].Instances[].InstanceId" \
  --output text

```
Output Example of InstanceId
```
i-0abc12345def67890

```
*Save InstanceId*

2. Create an AMI from the Instance devops-ec2-ami

```
aws ec2 create-image \
  --instance-id <your-instance-id> \
  --name devops-ec2-ami \
  --description "AMI created from devops-ec2 instance" \
  --no-reboot

```
*Outputs the ImageId, save it*

3. Wait Until the AMI Is in available State

```
aws ec2 wait image-available \
  --image-ids <your-imageId>

```
*Returns No output, which means the AMI is now available.*

4. Verify AMI state
```
aws ec2 describe-images \
  --image-ids <your-imageId> \
  --query "Images[].State" \
  --output text

```

**Returns**
```
available
```

