# Task 18
> When establishing infrastructure on the AWS cloud, Identity and Access Management (IAM) is among the first and most critical services to configure. IAM facilitates the creation and management of user accounts, groups, roles, policies, and other access controls. 
> The Nautilus DevOps team is currently in the process of configuring these resources and has outlined the following requirements.

> Create an IAM policy named iampolicy_mark in us-east-1 region, it must allow read-only access to the EC2 console, i.e this policy must allow users to view all instances, AMIs, and snapshots in the Amazon EC2 console.


### Create IAM policy iampolicy_mark (EC2 Read-Only)


1. Load AWS credentials
On the aws-client host, first get the credentials:
```
showcreds
```
Then configure AWS CLI
```
aws configure
```

Enter the values exactly as provided:

AWS Access Key ID

AWS Secret Access Key

Default region name: us-east-1

Default output format: json

> Note: The Kodekloud lab is already configured

2. Create the IAM policy document
Create a file called ec2-readonly-policy.json:
```
vi ec2-readonly-policy.json
```
Add this content to file
```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeInstances",
        "ec2:DescribeImages",
        "ec2:DescribeSnapshots",
        "ec2:DescribeVolumes",
        "ec2:DescribeSecurityGroups",
        "ec2:DescribeKeyPairs",
        "ec2:DescribeAvailabilityZones",
        "ec2:DescribeTags"
      ],
      "Resource": "*"
    }
  ]
}

```

3. Create IAM Policy 
```
aws iam create-policy \
  --policy-name iampolicy_mark \
  --policy-document file://ec2-readonly-policy.json

```
4. Verify if Policy exists
```
aws iam list-policies --scope Local | grep iampolicy_mark

```


