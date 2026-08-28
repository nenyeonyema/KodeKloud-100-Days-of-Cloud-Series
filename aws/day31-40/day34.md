# AWS Task 34 — Create an AWS Lambda Function Using AWS CLI

## 📌 Task Overview

The Nautilus DevOps Team required an automated, programmatic deployment of a serverless execution environment to transition from manual AWS Console workflows to Infrastructure as Code (IaC) scripting. The objective was to use the AWS CLI to create an IAM execution role (`lambda_execution_role`), attach the AWS-managed basic execution policy, package a Python function handler into a deployment zip archive, create an AWS Lambda function named `datacenter-lambda`, and validate execution via CLI invocation in `us-east-1`.

| Requirement | Value |
| :--- | :--- |
| **Function Name** | `datacenter-lambda` |
| **Runtime** | Python (`python3.12` / `python3.11`) |
| **Handler** | `lambda_function.lambda_handler` |
| **IAM Execution Role** | `lambda_execution_role` |
| **Attached Policy** | `arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole` |
| **Response Body** | `"Welcome to KKE AWS Labs!"` |
| **Response Status Code**| `200` |
| **AWS Region Context** | `us-east-1` |

---

## 🎯 Architecture & Automation Workflow

```text
                           AWS CLI Automation
                                   │
                                   ├───► 1. Create IAM Role
                                   │     (lambda_execution_role)
                                   │
                                   ├───► 2. Attach Managed Policy
                                   │     (AWSLambdaBasicExecutionRole)
                                   │
                                   ├───► 3. Zip Handler Script
                                   │     (function.zip)
                                   │
                                   ▼
                        aws lambda create-function
                                   │
                                   ▼
                         ┌──────────────────┐
                         │    AWS Lambda    │
                         │ datacenter-lambda│
                         └────────┬─────────┘
                                  │
                                  │ aws lambda invoke
                                  ▼
                         ┌──────────────────┐
                         │ lambda_handler() │
                         │   Python 3.12    │
                         └────────┬─────────┘
                                  │
                                  ▼
                    ┌─────────────────────────────┐
                    │ Response Payload:           │
                    │ {                           │
                    │   "statusCode": 200,        │
                    │   "body": "Welcome to KKE..."│
                    │ }                           │
                    └─────────────────────────────┘
🛠️ Implementation & Verification Commands
Bash
# ==========================================
# 1. CONFIGURE REGION & CREATE IAM EXECUTION ROLE
# ==========================================
aws configure set region us-east-1

# Create Execution Role with Service Trust Policy
aws iam create-role \
  --role-name lambda_execution_role \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Principal": {
          "Service": "lambda.amazonaws.com"
        },
        "Action": "sts:AssumeRole"
      }
    ]
  }'

# Attach AWS-Managed Basic Execution Policy (CloudWatch Logs)
aws iam attach-role-policy \
  --role-name lambda_execution_role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

# Resolve and Store IAM Role ARN
ROLE_ARN=$(aws iam get-role \
  --role-name lambda_execution_role \
  --query "Role.Arn" \
  --output text)

echo "IAM Role ARN: ${ROLE_ARN}"

# Allow IAM policy propagation delay
sleep 10

# ==========================================
# 2. WRITE HANDLER & BUILD ZIP PACKAGE
# ==========================================
cat > lambda_function.py <<'EOF'
def lambda_handler(event, context):
    return {
        "statusCode": 200,
        "body": "Welcome to KKE AWS Labs!"
    }
EOF

# Create Zip Archive
zip function.zip lambda_function.py

# ==========================================
# 3. CREATE LAMBDA FUNCTION & AWAIT ACTIVE STATE
# ==========================================
aws lambda create-function \
  --function-name datacenter-lambda \
  --runtime python3.12 \
  --role "$ROLE_ARN" \
  --handler lambda_function.lambda_handler \
  --zip-file fileb://function.zip \
  --region us-east-1

# Verify Active Function State
aws lambda get-function \
  --function-name datacenter-lambda \
  --query "Configuration.{State:State, LastUpdateStatus:LastUpdateStatus}" \
  --output table \
  --region us-east-1

# ==========================================
# 4. INVOKE FUNCTION & VERIFY RESPONSE
# ==========================================
# Invoke via CLI using raw-in-base64-out formatting
aws lambda invoke \
  --function-name datacenter-lambda \
  --payload '{}' \
  --cli-binary-format raw-in-base64-out \
  response.json \
  --region us-east-1

# Display Returned Response Payload
cat response.json

# Verify Detailed Function Configuration
aws lambda get-function-configuration \
  --function-name datacenter-lambda \
  --query "{Function:FunctionName, Runtime:Runtime, Handler:Handler, Role:Role, State:State}" \
  --output table \
  --region us-east-1
Note: Executing cat response.json returning "statusCode": 200 and "body": "Welcome to KKE AWS Labs!" confirms that datacenter-lambda was successfully built, packaged, deployed, and invoked entirely via the AWS CLI.
