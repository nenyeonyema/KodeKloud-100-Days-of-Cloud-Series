
# Task 47 — Priority Queuing with SQS, SNS, and Lambda via CloudFormation

## Overview

This task implements a priority-based message queuing system using Amazon SQS and SNS, deployed entirely through AWS CloudFormation. Messages published to an SNS topic are automatically routed to either a high-priority or low-priority SQS queue based on message attributes. A Lambda function consumes messages from both queues and processes high-priority messages first.

---

## Architecture

```
Publisher
    ↓
SNS Topic (devops-Priority-Queues-Topic)
    ↓ [Filter Policy: priority=high]        ↓ [Filter Policy: priority=low]
High Priority Queue                     Low Priority Queue
(devops-High-Priority-Queue)            (devops-Low-Priority-Queue)
    ↓                                       ↓
        Lambda Function (devops-priorities-queue-function)
                    ↓
            CloudWatch Logs
```

---

## Resources Created

| Resource | Name | Description |
|---|---|---|
| SQS Queue | `devops-High-Priority-Queue` | Receives high priority messages |
| SQS Queue | `devops-Low-Priority-Queue` | Receives low priority messages |
| SNS Topic | `devops-Priority-Queues-Topic` | Entry point for all messages |
| Lambda Function | `devops-priorities-queue-function` | Processes messages in priority order |
| IAM Role | `lambda_execution_role` | Execution role for Lambda |
| SNS Subscription (High) | Filter: `priority=high` | Routes high priority to high queue |
| SNS Subscription (Low) | Filter: `priority=low` | Routes low priority to low queue |

---

## Prerequisites

- AWS CLI configured with appropriate credentials
- Region: `us-east-1`
- CloudFormation stack name: `devops-priority-stack`
- Template path on aws-client: `/root/devops-priority-stack.yml`

---

## CloudFormation Template

**File:** `devops-priority-stack.yml`

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: Nautilus DevOps - Priority Queuing with SQS, SNS, and Lambda

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
      Policies:
        - PolicyName: SQSReceivePolicy
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - sqs:ReceiveMessage
                  - sqs:DeleteMessage
                  - sqs:GetQueueAttributes
                  - sqs:ChangeMessageVisibility
                Resource:
                  - !GetAtt HighPriorityQueue.Arn
                  - !GetAtt LowPriorityQueue.Arn
        - PolicyName: SNSPublishPolicy
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - sns:Publish
                  - sns:Subscribe
                  - sns:ListTopics
                Resource: !Ref PriorityTopic
        - PolicyName: CloudWatchLogsPolicy
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - logs:CreateLogGroup
                  - logs:CreateLogStream
                  - logs:PutLogEvents
                Resource: arn:aws:logs:*:*:*

  # SQS Queues
  HighPriorityQueue:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: devops-High-Priority-Queue
      VisibilityTimeout: 60
      MessageRetentionPeriod: 86400

  LowPriorityQueue:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: devops-Low-Priority-Queue
      VisibilityTimeout: 60
      MessageRetentionPeriod: 86400

  # SQS Queue Policies (allow SNS to send messages)
  HighPriorityQueuePolicy:
    Type: AWS::SQS::QueuePolicy
    Properties:
      Queues:
        - !Ref HighPriorityQueue
      PolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: sns.amazonaws.com
            Action: sqs:SendMessage
            Resource: !GetAtt HighPriorityQueue.Arn
            Condition:
              ArnEquals:
                aws:SourceArn: !Ref PriorityTopic

  LowPriorityQueuePolicy:
    Type: AWS::SQS::QueuePolicy
    Properties:
      Queues:
        - !Ref LowPriorityQueue
      PolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: sns.amazonaws.com
            Action: sqs:SendMessage
            Resource: !GetAtt LowPriorityQueue.Arn
            Condition:
              ArnEquals:
                aws:SourceArn: !Ref PriorityTopic

  # SNS Topic
  PriorityTopic:
    Type: AWS::SNS::Topic
    Properties:
      TopicName: devops-Priority-Queues-Topic

  # SNS Subscriptions with filter policies
  HighPrioritySubscription:
    Type: AWS::SNS::Subscription
    Properties:
      TopicArn: !Ref PriorityTopic
      Protocol: sqs
      Endpoint: !GetAtt HighPriorityQueue.Arn
      FilterPolicy:
        priority:
          - high
      RawMessageDelivery: false

  LowPrioritySubscription:
    Type: AWS::SNS::Subscription
    Properties:
      TopicArn: !Ref PriorityTopic
      Protocol: sqs
      Endpoint: !GetAtt LowPriorityQueue.Arn
      FilterPolicy:
        priority:
          - low
      RawMessageDelivery: false

  # Lambda Function
  PriorityQueueFunction:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: devops-priorities-queue-function
      Runtime: python3.12
      Handler: index.lambda_handler
      Role: !GetAtt LambdaExecutionRole.Arn
      Timeout: 60
      Code:
        ZipFile: |
          import json
          import logging

          logger = logging.getLogger()
          logger.setLevel(logging.INFO)

          def lambda_handler(event, context):
              high_priority_messages = []
              low_priority_messages = []

              for record in event.get('Records', []):
                  body = record.get('body', '')
                  queue_arn = record.get('eventSourceARN', '')

                  try:
                      msg = json.loads(body)
                      message_text = msg.get('Message', body)
                  except Exception:
                      message_text = body

                  if 'High-Priority-Queue' in queue_arn:
                      high_priority_messages.append(message_text)
                  else:
                      low_priority_messages.append(message_text)

              logger.info("=== Processing Order ===")
              for msg in high_priority_messages:
                  logger.info(f"[HIGH PRIORITY] {msg}")
              for msg in low_priority_messages:
                  logger.info(f"[LOW PRIORITY]  {msg}")

              return {
                  'statusCode': 200,
                  'body': json.dumps({
                      'high_priority_processed': len(high_priority_messages),
                      'low_priority_processed': len(low_priority_messages)
                  })
              }

  # Lambda Event Source Mappings
  HighPriorityEventSource:
    Type: AWS::Lambda::EventSourceMapping
    Properties:
      EventSourceArn: !GetAtt HighPriorityQueue.Arn
      FunctionName: !GetAtt PriorityQueueFunction.Arn
      BatchSize: 10
      Enabled: true
      FunctionResponseTypes:
        - ReportBatchItemFailures

  LowPriorityEventSource:
    Type: AWS::Lambda::EventSourceMapping
    Properties:
      EventSourceArn: !GetAtt LowPriorityQueue.Arn
      FunctionName: !GetAtt PriorityQueueFunction.Arn
      BatchSize: 10
      Enabled: true
      FunctionResponseTypes:
        - ReportBatchItemFailures

Outputs:
  HighPriorityQueueURL:
    Description: URL of the High Priority SQS Queue
    Value: !Ref HighPriorityQueue

  LowPriorityQueueURL:
    Description: URL of the Low Priority SQS Queue
    Value: !Ref LowPriorityQueue

  SNSTopicARN:
    Description: ARN of the SNS Topic
    Value: !Ref PriorityTopic

  LambdaFunctionARN:
    Description: ARN of the Lambda Function
    Value: !GetAtt PriorityQueueFunction.Arn
```

---

## Deployment Instructions

### Step 1 — Validate the template

```bash
aws cloudformation validate-template \
  --template-body file:///root/devops-priority-stack.yml \
  --region us-east-1
```

### Step 2 — Deploy the stack

```bash
aws cloudformation deploy \
  --template-file /root/devops-priority-stack.yml \
  --stack-name devops-priority-stack \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1 \
  --no-fail-on-empty-changeset
```

### Step 3 — Verify stack outputs

```bash
aws cloudformation describe-stacks \
  --stack-name devops-priority-stack \
  --query 'Stacks[0].Outputs' \
  --output table
```

---

## Testing

### Publish test messages to SNS

```bash
# Get topic ARN
topicarn=$(aws sns list-topics \
  --query "Topics[?contains(TopicArn,'devops-Priority-Queues-Topic')].TopicArn" \
  --output text)

# Publish high priority messages
aws sns publish --topic-arn $topicarn \
  --message 'High Priority message 1' \
  --message-attributes '{"priority":{"DataType":"String","StringValue":"high"}}'

aws sns publish --topic-arn $topicarn \
  --message 'High Priority message 2' \
  --message-attributes '{"priority":{"DataType":"String","StringValue":"high"}}'

# Publish low priority messages
aws sns publish --topic-arn $topicarn \
  --message 'Low Priority message 1' \
  --message-attributes '{"priority":{"DataType":"String","StringValue":"low"}}'

aws sns publish --topic-arn $topicarn \
  --message 'Low Priority message 2' \
  --message-attributes '{"priority":{"DataType":"String","StringValue":"low"}}'
```

### Verify Lambda processed messages

```bash
# Check CloudWatch logs
LOG_GROUP="/aws/lambda/devops-priorities-queue-function"

LOG_STREAM=$(aws logs describe-log-streams \
  --log-group-name $LOG_GROUP \
  --order-by LastEventTime --descending \
  --max-items 1 \
  --query 'logStreams[0].logStreamName' \
  --output text)

aws logs get-log-events \
  --log-group-name $LOG_GROUP \
  --log-stream-name "$LOG_STREAM" \
  --query 'events[*].message' \
  --output text
```

Expected log output:
```
=== Processing Order ===
[HIGH PRIORITY] High Priority message 1
[HIGH PRIORITY] High Priority message 2
[LOW PRIORITY]  Low Priority message 1
[LOW PRIORITY]  Low Priority message 2
```

### Check queue depths

```bash
# High priority queue — should be 0 after Lambda consumed
HIGH_URL=$(aws cloudformation describe-stacks \
  --stack-name devops-priority-stack \
  --query "Stacks[0].Outputs[?OutputKey=='HighPriorityQueueURL'].OutputValue" \
  --output text)

aws sqs get-queue-attributes \
  --queue-url $HIGH_URL \
  --attribute-names ApproximateNumberOfMessages

# Low priority queue
LOW_URL=$(aws cloudformation describe-stacks \
  --stack-name devops-priority-stack \
  --query "Stacks[0].Outputs[?OutputKey=='LowPriorityQueueURL'].OutputValue" \
  --output text)

aws sqs get-queue-attributes \
  --queue-url $LOW_URL \
  --attribute-names ApproximateNumberOfMessages
```

---

## How Priority Routing Works

### SNS Filter Policies
The key mechanism is SNS message filtering on subscriptions:

| Message Attribute | Routed To |
|---|---|
| `priority = high` | `devops-High-Priority-Queue` |
| `priority = low` | `devops-Low-Priority-Queue` |
[O| No attribute | Dropped (not delivered to either queue) |

### Lambda Priority Processing
The Lambda function has two separate event source mappings — one per queue. It identifies which queue a message came from using `eventSourceARN` and always logs high-priority messages before low-priority ones within the same batch.

```python
if 'High-Priority-Queue' in queue_arn:
    high_priority_messages.append(message_text)
else:
    low_priority_messages.append(message_text)

# High priority always processed first
for msg in high_priority_messages:
    logger.info(f"[HIGH PRIORITY] {msg}")
for msg in low_priority_messages:
    logger.info(f"[LOW PRIORITY]  {msg}")
```

---

## IAM Permissions Summary

| Policy | Permissions | Resource |
|---|---|---|
| SQSReceivePolicy | `sqs:ReceiveMessage`, `sqs:DeleteMessage`, `sqs:GetQueueAttributes`, `sqs:ChangeMessageVisibility` | Both SQS queues |
| SNSPublishPolicy | `sns:Publish`, `sns:Subscribe`, `sns:ListTopics` | SNS topic |
| CloudWatchLogsPolicy | `logs:CreateLogGroup`, `logs:CreateLogStream`, `logs:PutLogEvents` | All log groups |

---

## Key Concepts

### Why SQS Queue Policy is needed
SNS cannot deliver messages to SQS without an explicit `sqs:SendMessage` permission on the queue. The queue policy grants this with a condition tied to the specific SNS topic ARN to prevent unauthorized access.

### Why CAPABILITY_NAMED_IAM is required
The CloudFormation template creates an IAM role with a specific name (`lambda_execution_role`). AWS requires explicit acknowledgement via `CAPABILITY_NAMED_IAM` whenever a stack creates named IAM resources.

### RawMessageDelivery: false
Keeps the SNS envelope around the message body. This allows the Lambda to parse the `Message` field and extract the original message text from the SNS wrapper JSON.

---

## Cleanup

```bash
aws cloudformation delete-stack \
  --stack-name devops-priority-stack \
  --region us-east-1

aws cloudformation wait stack-delete-complete \
  --stack-name devops-priority-stack
```

---

## Verification Checklist

- [ ] Stack status is `CREATE_COMPLETE`
- [ ] Both SQS queues exist and are empty after Lambda processes
- [ ] SNS topic exists with two subscriptions (high and low)
- [ ] Lambda function exists with two event source mappings
- [ ] High priority messages logged before low priority in CloudWatch
- [ ] Queue depths return to 0 after messages are published

---

## Author
Nenye — Cloud & DevOps Engineer
Stack: AWS SQS · SNS · Lambda · CloudFormation · IAM
Region: us-east-1
