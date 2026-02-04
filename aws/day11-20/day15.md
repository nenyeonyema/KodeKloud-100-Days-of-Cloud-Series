# Task 15
> The Nautilus DevOps team has some volumes in different regions in their AWS account. They are going to setup some automated backups so that all important data can be backed up on regular basis. For now they shared some requirements to take a snapshot of one of the volumes they have.
> Create a snapshot of an existing volume named datacenter-vol in us-east-1 region.
> 1. The name of the snapshot must be datacenter-vol-ss.
> 2. The description must be datacenter Snapshot.
> 3. Make sure the snapshot status is completed before submitting the task.
> Use below given AWS Credentials: (You can run the showcreds command on aws-client host to retrieve these credentials) 

### Create a snapshot of an existing volume named datacenter-vol

1. Get the Volume ID for datacenter-vol
```
aws ec2 describe-volumes \
  --filters "Name=tag:Name,Values=datacenter-vol" \
  --query "Volumes[].VolumeId" \
  --output text

```
Save the VolumeId 

2. Create the Snapshot
*Name + Description must match exactly*
```
aws ec2 create-snapshot \
  --volume-id <your-volumeId> \
  --description "datacenter Snapshot"

```

Save SnapshotId

3. Tag the Snapshot with the Required Name
*Snapshots don’t get a Name automatically, you must tag it.*
```
aws ec2 create-tags \
  --resources <your-snapshotId> \
  --tags Key=Name,Value=datacenter-vol-ss

```

4. Wait Until Snapshot Status Is completed
```
aws ec2 wait snapshot-completed \
  --snapshot-ids <your-snapshotId>

```
*No output = snapshot creation completed successfully*

5. Verify Snapshot status
```
aws ec2 describe-snapshots \
  --snapshot-ids <your-snapshotId> \
  --query "Snapshots[].State" \
  --output text

```

