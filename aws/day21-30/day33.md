# AWS Task 33 — Create an AWS Lambda Function

## 📌 Task Overview

The Nautilus DevOps Team required a basic serverless execution test using AWS Lambda to validate event-driven compute capabilities. The objective was to construct an AWS Lambda function named `datacenter-lambda` using the Python runtime, associate it with an IAM execution role (`lambda_execution_role`), deploy the function code returning an HTTP 200 payload with a custom greeting message, and verify execution via test events in `us-east-1`.

| Requirement | Value |
| :--- | :--- |
| **Function Name** | `datacenter-lambda` |
| **Runtime** | Python (`python3.12` / `python3.11`) |
| **IAM Execution Role** | `lambda_execution_role` |
| **IAM Policy** | `AWSLambdaBasicExecutionRole` (CloudWatch Logging) |
| **Response Body** | `"Welcome to KKE AWS Labs!"` |
| **Response Status Code**| `200` |
| **AWS Region Context** | `us-east-1` |

---

## 🎯 Architecture & Execution Boundary

```text
               Event Trigger / Manual Invocation
                              │
                              ▼
                     ┌─────────────────┐
                     │   AWS Lambda    │
                     │datacenter-lambda│
                     └────────┬────────┘
                              │ Assumes Role
                              ▼
                     ┌─────────────────┐
                     │    IAM Role     │
                     │lambda_execution_│
                     │      role       │
                     └────────┬────────┘
                              │
                              ▼
                     ┌─────────────────┐
                     │ lambda_handler()│
                     │  Python Engine  │
                     └────────┬────────┘
                              │
                              ▼
               ┌─────────────────────────────┐
               │ Return JSON Payload:        │
               │ {                           │
               │   "statusCode": 200,        │
               │   "body": "Welcome to KKE..."│
               │ }                           │
               └─────────────────────────────┘
🛠️ Implementation & Verification Commands
Bash
# ==========================================
# 1. CONFIGURE REGION & PREPARE IAM ROLE
# ==========================================
aws configure set region us-east-1

# Create Trust Policy for AWS Lambda Service
cat > trust-policy.json <<'EOF'
{
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
}
EOF

# Create Execution Role
ROLE_ARN=$(aws iam create-role \
  --role-name lambda_execution_role \
  --assume-role-policy-document file://trust-policy.json \
  --query "Role.Arn" \
  --output text)

# Attach Basic Execution Policy for CloudWatch Logs
aws iam attach-role-policy \
  --role-name lambda_execution_role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

# Wait briefly for IAM propagation
sleep 10

# ==========================================
# 2. CREATE FUNCTION DEPLOYMENT PACKAGE
# ==========================================
cat > lambda_function.py <<'EOF'
def lambda_handler(event, context):
    return {
        "statusCode": 200,
        "body": "Welcome to KKE AWS Labs!"
    }
EOF

zip function.zip lambda_function.py

# ==========================================
# 3. CREATE & DEPLOY LAMBDA FUNCTION
# ==========================================
aws lambda create-function \
  --function-name datacenter-lambda \
  --runtime python3.12 \
  --role "$ROLE_ARN" \
  --handler lambda_function.lambda_handler \
  --zip-file fileb://function.zip \
  --region us-east-1

# ==========================================
# 4. TEST FUNCTION & VERIFY PAYLOAD
# ==========================================
# Invoke Lambda Function with Empty Payload
aws lambda invoke \
  --function-name datacenter-lambda \
  --payload '{}' \
  response.json \
  --region us-east-1

# Display Execution Output
cat response.json
Note: A returned JSON response containing "statusCode": 200 and "body": "Welcome to KKE AWS Labs!" confirms that datacenter-lambda was successfully deployed, assumed the lambda_execution_role, and executed the Python handler as intended.
