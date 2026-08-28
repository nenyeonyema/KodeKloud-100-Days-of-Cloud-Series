# AWS Task 33 — Create an AWS Lambda Function via Console

## 📌 Task Overview

The Nautilus DevOps Team needed a simple serverless function to demonstrate AWS Lambda. The function returns a custom greeting with an HTTP-style status code of 200. The task was completed using the AWS Console in the us-east-1 region.

| Requirement | Target Value |
| :--- | :--- |
| **Deployment Method** | AWS Management Console |
| **Function Name** | `datacenter-lambda` |
| **Runtime** | Python |
| **IAM Execution Role** | `lambda_execution_role` |
| **Attached Managed Policy**| `AWSLambdaBasicExecutionRole` |
| **Response Body** | `"Welcome to KKE AWS Labs!"` |
| **Response Status Code**| `200` |
| **AWS Region Context** | `us-east-1` (US East - N. Virginia) |

---

## 🎯 How It Works

```text
                AWS Lambda
                    |
                    v
          +-------------------+
          | datacenter-lambda |
          |                   |
          | Python Runtime    |
          +---------+---------+
                    |
                    v
          lambda_handler()
                    |
                    v
        +----------------------+
        | statusCode: 200      |
        |                      |
        | Welcome to KKE       |
        | AWS Labs!            |
        +----------------------+
IAM Execution Role
The Lambda function uses:

lambda_execution_role

The execution role gives Lambda permission to interact with AWS services when required. For a basic Lambda function, the role typically includes the AWS-managed policy:

AWSLambdaBasicExecutionRole

This allows the function to write execution logs to Amazon CloudWatch Logs.

The important relationship is:

Plaintext
Lambda Function
      |
      v
IAM Execution Role
      |
      v
AWS Permissions
      |
      v
AWS Services
Lambda does not receive AWS permissions simply because the function exists. The execution role determines what the function is allowed to do.

📋 Step-by-Step Execution Guide
1. Open AWS Lambda
From the AWS Management Console:

Select the Lambda service.

Ensure the region is US East (N. Virginia) — us-east-1.

Click Create function.

2. Create the Lambda Function
Select:

Author from scratch

Function name: datacenter-lambda

Runtime: Python

Under permissions, create a new execution role:

Role name: lambda_execution_role

Allow Lambda to create the required basic execution permissions.

Click Create function.

3. Configure the Function Code
In the Lambda code editor, use:

Python
def lambda_handler(event, context):
    return {
        "statusCode": 200,
        "body": "Welcome to KKE AWS Labs!"
    }
Click Deploy to save the function code.

4. Test the Lambda Function
Click Test and create a test event.

A simple JSON event can be used:

JSON
{}
Run the test.

The expected response is:

JSON
{
  "statusCode": 200,
  "body": "Welcome to KKE AWS Labs!"
}
The successful response confirms that the Lambda function is executing correctly.

💡 Key Learning
This task demonstrated the basic serverless deployment model with AWS Lambda.

Unlike an EC2-based application, there is no server to provision or maintain. AWS manages the underlying compute infrastructure and executes the function when it is invoked.

The basic workflow is:

Plaintext
Create Function
      ↓
Choose Runtime
      ↓
Configure IAM Role
      ↓
Deploy Code
      ↓
Invoke Function
      ↓
Verify Response
✅ Final Verification
[x] Created datacenter-lambda

[x] Used Python runtime

[x] Created lambda_execution_role

[x] Configured the Lambda execution role

[x] Returned Welcome to KKE AWS Labs!

[x] Configured status code 200

[x] Deployed the function

[x] Tested the Lambda function successfully

[x] Used us-east-1
