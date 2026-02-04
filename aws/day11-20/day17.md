# Task 17
> The Nautilus DevOps team has been creating a couple of services on AWS cloud. They have been breaking down the migration into smaller tasks, allowing for better control, risk mitigation, and optimization of resources throughout the migration process. Recently they came up with requirements mentioned below.

> Create an IAM group named iamgroup_ravi.

### Create an IAM Group named iamgroup_ravi

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

2. Create the IAM user
```
aws iam create-group --group-name iamgroup_ravi
```

If successful returns similar output like below
```
{
  "Group": {
    "GroupName": "iamgroup_ravi",
    "Arn": "arn:aws:iam::ACCOUNT_ID:group/iamgroup_ravi",
    "CreateDate": "2026-02-03T..."
  }
}

```
3. Verify if group exists
```
aws iam get-group --group-name iamgroup_ravi
```
Returns similar output like we have above

