# Task 16
> When establishing infrastructure on the AWS cloud, Identity and Access Management (IAM) is among the first and most critical services to configure. IAM facilitates the creation and management of user accounts, groups, roles, policies, and other access controls.
> The Nautilus DevOps team is currently in the process of configuring these resources and has outlined the following requirements:

> For this task, create an IAM user named iamuser_rose.

### Create an IAM User named iamuser_rose

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

> The Kodekloud lab is already configured

2. Create the IAM user
```
aws iam create-user --user-name iamuser_rose
```

If successful returns similar output like below
```
{
  "User": {
    "UserName": "iamuser_rose",
    "Arn": "arn:aws:iam::ACCOUNT_ID:user/iamuser_rose",
    "CreateDate": "2026-02-02T..."
  }
}

```
3. Verify if user exist
```
aws iam get-user --user-name iamuser_rose
```
Returns similar output like we have above

