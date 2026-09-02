# Task 46 — Automated S3 File Transfer with Lambda and DynamoDB Logging

## Overview

This task automates secure file transfer between two S3 buckets using an event-driven serverless architecture. A public S3 bucket acts as an intake point for file uploads. Whenever a file is uploaded, a Lambda function is automatically triggered to copy it to a private S3 bucket and log the transfer details to a DynamoDB table.

---

## Architecture

```
Upload → Public S3 Bucket
              ↓ (S3 ObjectCreated Event)
         Lambda Function
         ↙              ↘
Private S3 Bucket    DynamoDB Table
(file copy)          (transfer log)
```

---

## Resources Created

| Resource | Name | Description |
|---|---|---|
| S3 Bucket (Public) | `datacenter-public-25497` | Intake bucket for file uploads |
| S3 Bucket (Private) | `datacenter-private-1920` | Secure storage for copied files |
| Lambda Function | `datacenter-copyfunction` | Copies file and logs transfer |
| IAM Role | `lambda_execution_role` | Execution role for Lambda |
| DynamoDB Table | `datacenter-S3CopyLogs` | Stores transfer audit logs |

---

## Prerequisites

- AWS CLI configured with appropriate credentials
- Region: `us-east-1`
- Python 3.12 runtime support

---

## Setup Instructions

### Step 1 — Create Public S3 Bucket

```bash
aws s3api create-bucket \
  --bucket datacenter-public-25497 \
  --region us-east-1

# Disable block public access
aws s3api put-public-access-block \
  --bucket datacenter-public-25497 \
  --public-access-block-configuration \
  "BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=false,RestrictPublicBuckets=false"

# Apply public read policy
aws s3api put-bucket-policy \
  --bucket datacenter-public-25497 \
  --policy '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::datacenter-public-25497/*"
    }]
  }'
```

### Step 2 — Create Private S3 Bucket

```bash
aws s3api create-bucket \
  --bucket datacenter-private-1920 \
  --region us-east-1

# Block all public access
aws s3api put-public-access-block \
  --bucket datacenter-private-1920 \
  --public-access-block-configuration \
  "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"
```

### Step 3 — Create DynamoDB Table

```bash
aws dynamodb create-table \
  --table-name datacenter-S3CopyLogs \
  --attribute-definitions AttributeName=LogID,AttributeType=S \
  --key-schema AttributeName=LogID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region us-east-1

# Wait for table to be active
aws dynamodb wait table-exists --table-name datacenter-S3CopyLogs
```

### Step 4 — Create IAM Role and Policies

```bash
# Create role with Lambda trust policy
aws iam create-role \
  --role-name lambda_execution_role \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "lambda.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }]
  }'

# S3 read/write policy
aws iam put-role-policy \
  --role-name lambda_execution_role \
  --policy-name S3CopyPolicy \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": ["s3:GetObject"],
        "Resource": "arn:aws:s3:::datacenter-public-25497/*"
      },
      {
        "Effect": "Allow",
        "Action": ["s3:PutObject"],
        "Resource": "arn:aws:s3:::datacenter-private-1920/*"
      }
    ]
  }'

# DynamoDB logging policy
aws iam put-role-policy \
  --role-name lambda_execution_role \
  --policy-name DynamoDBLoggingPolicy \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Action": ["dynamodb:PutItem"],
      "Resource": "arn:aws:dynamodb:us-east-1:*:table/datacenter-S3CopyLogs"
    }]
  }'

# CloudWatch Logs policy
aws iam attach-role-policy \
  --role-name lambda_execution_role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
[O```

### Step 5 — Lambda Function Code

```python
# lambda-function.py
import boto3
import uuid
import json
from datetime import datetime

s3_client = boto3.client('s3')
dynamodb = boto3.resource('dynamodb')

DYNAMODB_TABLE = 'datacenter-S3CopyLogs'
PRIVATE_BUCKET = 'datacenter-private-1920'

def lambda_handler(event, context):
    for record in event['Records']:
        source_bucket = record['s3']['bucket']['name']
        object_key = record['s3']['object']['key']

        print(f"Copying {object_key} from {source_bucket} to {PRIVATE_BUCKET}")

        # Copy file to private bucket
        copy_source = {'Bucket': source_bucket, 'Key': object_key}
        s3_client.copy_object(
            CopySource=copy_source,
            Bucket=PRIVATE_BUCKET,
            Key=object_key
        )

        # Log to DynamoDB
        table = dynamodb.Table(DYNAMODB_TABLE)
        log_item = {
            'LogID': str(uuid.uuid4()),
            'SourceBucket': source_bucket,
            'DestinationBucket': PRIVATE_BUCKET,
            'ObjectKey': object_key,
            'Timestamp': datetime.utcnow().isoformat()
        }
        table.put_item(Item=log_item)
        print(f"Logged: {json.dumps(log_item)}")

    return {'statusCode': 200, 'body': 'File copied and logged successfully'}
```

### Step 6 — Deploy Lambda Function

```bash
# Package the function
zip -j lambda_package.zip lambda-function.py

# Get role ARN
ROLE_ARN=$(aws iam get-role \
  --role-name lambda_execution_role \
  --query 'Role.Arn' --output text)

# Deploy
aws lambda create-function \
  --function-name datacenter-copyfunction \
  --runtime python3.12 \
  --role $ROLE_ARN \
  --handler lambda-function.lambda_handler \
  --zip-file fileb://lambda_package.zip \
  --timeout 30 \
  --region us-east-1
```

### Step 7 — Add S3 Trigger

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Grant S3 permission to invoke Lambda
aws lambda add-permission \
  --function-name datacenter-copyfunction \
  --statement-id S3InvokeLambda \
  --action lambda:InvokeFunction \
  --principal s3.amazonaws.com \
  --source-arn arn:aws:s3:::datacenter-public-25497 \
  --source-account $ACCOUNT_ID

# Get Lambda ARN
LAMBDA_ARN=$(aws lambda get-function \
  --function-name datacenter-copyfunction \
  --query 'Configuration.FunctionArn' --output text)

# Configure S3 notification
aws s3api put-bucket-notification-configuration \
  --bucket datacenter-public-25497 \
  --notification-configuration "{
    \"LambdaFunctionConfigurations\": [{
      \"Id\": \"TriggerLambdaOnUpload\",
      \"LambdaFunctionArn\": \"${LAMBDA_ARN}\",
      \"Events\": [\"s3:ObjectCreated:*\"]
    }]
  }"
```

### Step 8 — Test

```bash
# Upload test file
aws s3 cp sample.zip s3://datacenter-public-25497/sample.zip

# Wait for Lambda to execute
sleep 15

# Verify file copied to private bucket
aws s3 ls s3://datacenter-private-1920/

# Verify DynamoDB log entry
aws dynamodb scan --table-name datacenter-S3CopyLogs --output table
```

---

## DynamoDB Log Schema

| Attribute | Type | Description |
|---|---|---|
| `LogID` | String (PK) | Unique UUID for each log entry |
| `SourceBucket` | String | Name of the public source bucket |
| `DestinationBucket` | String | Name of the private destination bucket |
| `ObjectKey` | String | Key (filename) of the copied object |
| `Timestamp` | String | UTC timestamp of the transfer |

---

## IAM Permissions Summary

| Policy | Permission | Resource |
|---|---|---|
| S3CopyPolicy | `s3:GetObject` | Public bucket |
| S3CopyPolicy | `s3:PutObject` | Private bucket |
| DynamoDBLoggingPolicy | `dynamodb:PutItem` | Log table |
| AWSLambdaBasicExecutionRole | CloudWatch Logs | All |

---

## Common Issues and Fixes

### Lambda not triggering after upload
**Cause:** S3 notification configured before Lambda resource-based policy was added.
**Fix:** Always add Lambda invoke permission first, then configure S3 notification.

```bash
# Correct order:
# 1. Create Lambda
# 2. Add resource-based policy (lambda:add-permission)
# 3. Configure S3 notification (s3api:put-bucket-notification-configuration)
# 4. Upload file to test
```

### File already in bucket not triggering Lambda
**Cause:** S3 only fires `ObjectCreated` on new PUT events.
**Fix:** Delete and re-upload the file.

```bash
aws s3 rm s3://datacenter-public-25497/sample.zip
aws s3 cp sample.zip s3://datacenter-public-25497/sample.zip
```

### Handler name error
**Cause:** Zip file named `lambda-function.py` requires handler `lambda-function.lambda_handler` (hyphen, not underscore).
**Fix:** Ensure handler name matches filename exactly.

---

## Verification Checklist

- [ ] Public bucket exists and allows public read
- [ ] Private bucket exists with all public access blocked
- [ ] DynamoDB table `datacenter-S3CopyLogs` is ACTIVE
- [ ] IAM role `lambda_execution_role` has all three policies attached
- [ ] Lambda function exists and has S3 resource-based policy
- [ ] S3 notification configuration points to correct Lambda ARN
- [ ] After upload — file appears in private bucket
- [ ] After upload — log entry appears in DynamoDB

---

## Author
Nenye — Cloud & DevOps Engineer  
Stack: AWS S3 · Lambda · DynamoDB · IAM  
Region: us-east-1
