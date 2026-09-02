# Task 48 — Lambda Function Deployment via CloudFormation

## Overview

This task deploys a simple AWS Lambda function using a CloudFormation stack.
The Lambda function returns a welcome message with an HTTP 200 status code.
All resources including the IAM execution role are provisioned and managed
through a single CloudFormation template, demonstrating infrastructure as
code best practices for serverless deployments.

---

## Architecture

```
CloudFormation Stack (datacenter-lambda-app)
        ↓ creates
┌─────────────────────────────────┐
│  IAM Role                       │
│  (lambda_execution_role)        │
│           ↓ attached to         │
│  Lambda Function                │
│  (datacenter-lambda)            │
│           ↓ returns             │
│  { statusCode: 200,             │
│    body: "Welcome to            │
│           KKE AWS Labs!" }      │
└─────────────────────────────────┘
        ↓ logs to
  CloudWatch Logs
```

---

## Resources Created

| Resource | Name | Description |
|---|---|---|
| CloudFormation Stack | `datacenter-lambda-app` | Stack managing all resources |
| Lambda Function | `datacenter-lambda` | Returns welcome message with status 200 |
| IAM Role | `lambda_execution_role` | Execution role with CloudWatch Logs access |

---

## Prerequisites

- AWS CLI configured with appropriate credentials
- Region: `us-east-1`
- Template path on aws-client: `/root/datacenter-lambda.yml`
- Stack name: `datacenter-lambda-app`

---

## CloudFormation Template

**File:** `/root/datacenter-lambda.yml`

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: Nautilus DevOps - datacenter-lambda CloudFormation stack

Resources:

  # IAM Role
  LambdaExecutionRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: lambda_execution_role
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: lambda.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

  # Lambda Function
  DatacenterLambda:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: datacenter-lambda
      Runtime: python3.12
      Handler: index.lambda_handler
      Role: !GetAtt LambdaExecutionRole.Arn
      Timeout: 30
      Code:
        ZipFile: |
          import json

          def lambda_handler(event, context):
              return {
                  'statusCode': 200,
                  'body': json.dumps('Welcome to KKE AWS Labs!')
              }

Outputs:
  LambdaFunctionName:
    Description: Name of the Lambda function
    Value: !Ref DatacenterLambda

  LambdaFunctionARN:
    Description: ARN of the Lambda function
    Value: !GetAtt DatacenterLambda.Arn

  IAMRoleARN:
    Description: ARN of the IAM execution role
    Value: !GetAtt LambdaExecutionRole.Arn
```

---

## Lambda Function Code

```python
import json

def lambda_handler(event, context):
    return {
        'statusCode': 200,
        'body': json.dumps('Welcome to KKE AWS Labs!')
    }
```

### Response

```json
{
    "statusCode": 200,
    "body": "\"Welcome to KKE AWS Labs!\""
}
```

---

## Deployment Instructions

### Step 1 — Validate the template

```bash
aws cloudformation validate-template \
  --template-body file:///root/datacenter-lambda.yml \
  --region us-east-1
```

Expected output confirms the template is valid:
```json
{
    "Parameters": [],
    "Description": "Nautilus DevOps - datacenter-lambda CloudFormation stack"
}
```

### Step 2 — Deploy the stack

```bash
aws cloudformation deploy \
  --template-file /root/datacenter-lambda.yml \
  --stack-name datacenter-lambda-app \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1 \
  --no-fail-on-empty-changeset
```

> `CAPABILITY_NAMED_IAM` is required because the template creates an IAM
> role with a specific name (`lambda_execution_role`).

### Step 3 — Verify stack resources

```bash
aws cloudformation describe-stack-resources \
  --stack-name datacenter-lambda-app \
  --query 'StackResources[*].{Resource:LogicalResourceId,Type:ResourceType,Status:ResourceStatus}' \
  --output table
```

Expected output:
```
-----------------------------------------------------------------------
|               DescribeStackResources                                |
+---------------------+-------------------------+--------------------+
|  Resource           |  Type                   |  Status            |
+---------------------+-------------------------+--------------------+
|  DatacenterLambda   |  AWS::Lambda::Function  |  CREATE_COMPLETE   |
|  LambdaExecutionRole|  AWS::IAM::Role         |  CREATE_COMPLETE   |
+---------------------+-------------------------+--------------------+
```

### Step 4 — View stack outputs

```bash
aws cloudformation describe-stacks \
  --stack-name datacenter-lambda-app \
  --query 'Stacks[0].Outputs' \
  --output table
```

---

## Testing

### Invoke the Lambda function

```bash
aws lambda invoke \
  --function-name datacenter-lambda \
  --region us-east-1 \
  --cli-binary-format raw-in-base64-out \
  --payload '{}' \
  response.json
```

Expected invocation response:
```json
{
    "StatusCode": 200,
    "ExecutedVersion": "$LATEST"
}
```

### Check the function response body

```bash
cat response.json
```

Expected:
```json
{"statusCode": 200, "body": "\"Welcome to KKE AWS Labs!\""}
```

---

## IAM Role Details

### Trust Policy
Allows the Lambda service to assume the role:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Service": "lambda.amazonaws.com"
    },
    "Action": "sts:AssumeRole"
  }]
}
```

### Attached Managed Policy

| Policy | ARN | Purpose |
|---|---|---|
| AWSLambdaBasicExecutionRole | `arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole` | CloudWatch Logs — create log groups, log streams, put log events |

---

## Key Concepts

### ZipFile vs S3 deployment
The template uses `Code: ZipFile` for inline code deployment. This is
suitable for simple functions under 4KB. For larger functions, code should
be uploaded to S3 and referenced via `S3Bucket` and `S3Key`.

```yaml
# Inline (this task — simple function)
Code:
  ZipFile: |
    import json
    def lambda_handler(event, context):
        return {'statusCode': 200, 'body': json.dumps('Hello')}

# S3 reference (for larger functions)
Code:
  S3Bucket: my-deployment-bucket
  S3Key: my-function.zip
```

### Handler naming convention
The handler value `index.lambda_handler` means:
- `index` — the filename without `.py` extension
- `lambda_handler` — the function name inside the file

```yaml
Handler: index.lambda_handler
# File: index.py
# Function: def lambda_handler(event, context)
```

If the file is named differently (e.g. `lambda-function.py`), the handler
must match: `lambda-function.lambda_handler` — including the hyphen.

### CAPABILITY_NAMED_IAM
Required whenever a CloudFormation stack creates IAM resources with
custom names. Without this flag the deploy command will fail with:

```
InsufficientCapabilitiesException: Requires capabilities: CAPABILITY_NAMED_IAM
```

---

## Cleanup

```bash
aws cloudformation delete-stack \
  --stack-name datacenter-lambda-app \
  --region us-east-1

aws cloudformation wait stack-delete-complete \
  --stack-name datacenter-lambda-app
echo "Stack deleted."
```

---

## Verification Checklist

- [ ] Template validates successfully
- [ ] Stack status is `CREATE_COMPLETE`
- [ ] Both resources show `CREATE_COMPLETE`
- [ ] Lambda invocation returns `StatusCode: 200`
- [ ] Response body contains `Welcome to KKE AWS Labs!`
- [ ] IAM role `lambda_execution_role` exists with correct trust policy

---

## Author
Nenye — Cloud & DevOps Engineer
Stack: AWS Lambda · CloudFormation · IAM
Region: us-east-1
