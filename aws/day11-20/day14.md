# Task 14
> During the migration process, several resources were created under the AWS account. Later on, some of these resources became obsolete as alternative solutions were implemented. Similarly, there is an instance that needs to be deleted as it is no longer in use.

> * Delete the ec2 instance named datacenter-ec2 present in us-east-1 region.

> * Before submitting your task, make sure instance is in terminated state.

### Delete an ec2 instance named datacenter-ec2


1. Get the InstanceId of datacenter-ec2

```
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=datacenter-ec2" \
  --query "Reservations[].Instances[].InstanceId" \
  --output text

```
Output Example of InstanceId
```
i-0abc12345def67890

```
*Save InstanceId*

2. Disable Termination Protection if Enabled
If termination protection is enabled, termination will fail.

```
aws ec2 modify-instance-attribute \
  --instance-id <your-instanceid> \
  --no-disable-api-termination

```

3. Terminate EC2 Instance

```
aws ec2 terminate-instances \
  --instance-ids <your-instanceId>

```

4. Wait Until the Instance Is Fully Terminated
```
aws ec2 wait instance-terminated \
  --instance-ids i-0abc12345def67890

```
Returns no output which means instance is terminated

5. To verify

```
aws ec2 describe-instances \
  --instance-ids <your-instanceId> \
  --query "Reservations[].Instances[].State.Name" \
  --output text

```

**Returns**
```
terminated
```

