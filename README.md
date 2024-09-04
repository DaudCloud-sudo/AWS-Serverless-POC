# Proof of Concept: Serverless Web Backend on AWS

This project demonstrates how to build a proof of concept for a serverless solution on AWS. The solution is designed for a customer selling cleaning supplies, requiring a scalable architecture that handles spikes in demand while ensuring decoupled application components. These steps are part of a lab in an AWS Solutions Architect Associate preparation course.

## Architecture Overview

The architecture consists of the following components:

1. **REST API (API Gateway)**: Receives incoming requests and places a database entry in the Amazon SQS queue.
2. **Amazon SQS**: Stores the data temporarily before triggering the first Lambda function.
3. **AWS Lambda (Function 1)**: Inserts the entry into a DynamoDB table.
4. **DynamoDB Streams**: Captures the new entry and invokes a second Lambda function.
5. **AWS Lambda (Function 2)**: Passes the database entry to Amazon SNS.
6. **Amazon SNS**: Sends a notification to a specified email address.

## Table of Contents
- [Introduction](#introduction)
- [Architecture Diagram](#architecture-diagram)
- [Setup](#setup)
  - [Task 1: Creating IAM Policies and Roles](#task-1-creating-iam-policies-and-roles)
  - [Task 2: Creating a DynamoDB Table](#task-2-creating-a-dynamodb-table)
  - [Task 3: Creating an SQS Queue](#task-3-creating-an-sqs-queue)
  - [Task 4: Creating a Lambda Function and Setting Up Triggers](#task-4-creating-a-lambda-function-and-setting-up-triggers)
  - [Task 5: Enabling DynamoDB Streams](#task-5-enabling-dynamodb-streams)
  - [Task 6: Creating an SNS Topic and Setting Up Subscriptions](#task-6-creating-an-sns-topic-and-setting-up-subscriptions)
  - [Task 7: Creating a Lambda Function to Publish a Message to the SNS Topic](#task-7-creating-a-lambda-function-to-publish-a-message-to-the-sns-topic)
  - [Task 8: Creating an API with Amazon API Gateway](#task-8-creating-an-api-with-amazon-api-gateway)
  - [Task 9: Testing the Architecture](#task-9-testing-the-architecture)
  - [Task 10: Cleaning Up](#task-10-cleaning-up)

## Introduction

This README provides step-by-step instructions to set up a serverless backend on AWS for a business dealing in cleaning supplies. The solution is scalable, cost-effective, and designed to handle unpredictable spikes in traffic by decoupling key components.

## Architecture Diagram

![image](https://github.com/user-attachments/assets/92b514f0-3a40-4ae2-ae66-c5a52bba2d9c)

## Setup

### Task 1: Creating IAM Policies and Roles

To follow best practices, create custom IAM policies and roles to grant limited permissions.

#### Step 1.1: Creating Custom IAM Policies

1. Sign in to the AWS Management Console.
2. Search for **IAM** and select **Policies**.
3. Click **Create policy** and use the following JSON scripts to create the necessary policies:

**Policy 1: Lambda-Write-DynamoDB**
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "dynamodb:PutItem",
                "dynamodb:DescribeTable"
            ],
            "Resource": "*"
        }
    ]
}
```
**Policy 2: Lambda-SNS-Publish**
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "sns:Publish",
                "sns:GetTopicAttributes",
                "sns:ListTopics"
            ],
            "Resource": "*"
        }
    ]
}
```
**Policy 3: Lambda-DynamoDBStreams-Read**
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "dynamodb:GetShardIterator",
                "dynamodb:DescribeStream",
                "dynamodb:ListStreams",
                "dynamodb:GetRecords"
            ],
            "Resource": "*"
        }
    ]
}
```
**Policy 4: Lambda-Read-SQS**
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "sqs:DeleteMessage",
                "sqs:ReceiveMessage",
                "sqs:GetQueueAttributes",
                "sqs:ChangeMessageVisibility"
            ],
            "Resource": "*"
        }
    ]
}
```
![image](https://github.com/user-attachments/assets/d839f646-c20a-4fcc-babf-203dd6c26ba3)

#### Step 1.2: Creating IAM Roles and Attaching Policies

1. In the IAM console, navigate to **Roles**.
2. Click Create role and configure the following roles:

**Role 1: Lambda-SQS-DynamoDB**
- Trusted entity type: **AWS service**
- Common use case: **Lambda**
- Attach policies: `Lambda-Write-DynamoDB`, `Lambda-Read-SQS`

**Role 2: Lambda-DynamoDBStreams-SNS**
- Trusted entity type: **AWS service**
- Common use case: **Lambda**
- Attach policies: `Lambda-SNS-Publish`, `Lambda-DynamoDBStreams-Read`

**Role 3: APIGateway-SQS**
- Trusted entity type: **AWS service**
- Common use case: **API Gateway**
- Attach policies: `AmazonAPIGatewayPushToCloudWatchLogs`

![image](https://github.com/user-attachments/assets/db702161-4d98-446e-af2f-d77a2d583337)

### Task 2: Creating a DynamoDB Table

1. In the AWS Management Console, search for **DynamoDB** and select **Create table**.
2. Configure the following settings:
  - Table name: `orders`
  - Partition key: `orderID` (String)
3. Keep other settings as default and click **Create table**.

![image](https://github.com/user-attachments/assets/5c2df964-bf40-4a98-b2de-bcdaebfe4483)

### Task 3: Creating an SQS Queue

1. Search for SQS in the AWS Management Console and select **Create queue**.
2. Configure the following settings:
  - Name: `POC-Queue`
  - Access Policy: Basic
  - Sender: Only the specified AWS accounts, IAM users and roles
    - Paste the ARN of the `APIGateway-SQS` role.
  - Receiver: **Only the specified AWS accounts, IAM users and roles**
    - Paste the ARN of the `Lambda-SQS-DynamoDB` role.
3. Click **Create queue**.

### Task 4: Creating a Lambda Function and Setting Up Triggers

#### Step 4.1: Creating the Lambda Function

1. In the AWS Management Console, search for Lambda and select **Create function**.
2. Configure the following settings:
  - Function name: `POC-Lambda-1`
  - Runtime: **Python 3.9**
  - Execution role: Use an existing role
  - Existing role: `Lambda-SQS-DynamoDB`
3. Click Create function.

#### Step 4.2: Setting Up SQS as a Trigger

1. In the Function overview section, click **Add trigger**.
2. Choose SQS as the trigger service and select `POC-Queue`.
3. Click **Add**.

#### Step 4.3: Adding and Deploying Function Code

1. In the Code tab, replace the default code with:
```
import boto3, uuid

client = boto3.resource('dynamodb')
table = client.Table("orders")

def lambda_handler(event, context):
    for record in event['Records']:
        payload = record["body"]
        table.put_item(Item={'orderID': str(uuid.uuid4()), 'order': payload})
```
2. Click *Deploy*.

#### Step 4.4: Testing the Lambda Function

1. In the Test tab, create a new event:
  - Event name: `POC-Lambda-Test-1`
  - Template: **SQS**
2. Save and run the test.
3. Verify the test result and confirm the entry in the DynamoDB table.

### Task 5: Enabling DynamoDB Streams

1. In the DynamoDB console, select the `orders` table.
2. Go to the Exports and streams tab.
3. Enable DynamoDB Streams with the New image view type.

### Task 6: Creating an SNS Topic and Setting Up Subscriptions

#### Step 6.1: Creating an SNS Topic

1. In the AWS Management Console, search for **SNS** and select **Create topic**.
2. Configure the following settings:
  - Name: `POC-Topic`
  - Type: **Standard**
3. Save the ARN of the created topic.

#### Step 6.2: Subscribing to Email Notifications

1. In the **Subscriptions** tab, click **Create subscription**.
2. Topic ARN: `POC-Topic` ARN
  - Protocol: Email
  - Endpoint: Your email address
3. Confirm the subscription via the email you receive.

### Task 7: Creating a Lambda Function to Publish a Message to the SNS Topic

#### Step 7.1: Creating the POC-Lambda-2 Function

1. Navigate to the AWS Management Console and search for **Lambda**.
2. Click **Create function** and configure the following settings:
   - **Function name**: `POC-Lambda-2`
   - **Runtime**: `Python 3.9`
   - **Existing role**: Select `Lambda-DynamoDBStreams-SNS`
3. Click **Create function**.

#### Step 7.2: Setting Up DynamoDB as a Trigger

1. In the **Function overview** section of the Lambda console, click **Add trigger**.
2. Choose **DynamoDB** from the list of services.
3. Select the `orders` table as the trigger source.
4. Click **Add** to finalize the trigger setup.

#### Step 7.3: Configuring the Lambda Function

1. In the **Code** tab of the Lambda console, replace the default code with the following:

    ```python
    import boto3, json

    client = boto3.client('sns')

    def lambda_handler(event, context):
        for record in event['Records']:
            message = json.dumps(record['dynamodb']['NewImage'], indent=4)
            client.publish(
                TopicArn='arn:aws:sns:your-region:your-account-id:POC-Topic',
                Message=message,
                Subject='New Order Notification'
            )
    ```

2. Replace `arn:aws:sns:your-region:your-account-id:POC-Topic` with the actual ARN of your SNS topic.
3. Click **Deploy** to save and deploy the function.

### Task 8: Creating an API with Amazon API Gateway

1. Search for **API Gateway** in the AWS Management Console and select **Create API**.
2. Configure the following settings:
   - **API name**: `POC-API`
   - **API Type**: `REST API`
3. In the **Resources** section, create a **POST** method.
4. Integrate this method with the `POC-Queue` in SQS.
5. Deploy the API.

### Task 9: Testing the Architecture

1. Use **Postman** or **cURL** to send a POST request to your API Gateway endpoint.
2. Verify that a new entry is created in the DynamoDB table.
3. Check that an email notification is sent via SNS.

### Task 10: Cleaning Up

To avoid unnecessary charges, delete the following resources:

- Lambda functions
- SQS queue
- DynamoDB table and streams
- SNS topic
- API Gateway API
- IAM policies and roles

This serverless architecture provides a scalable solution for processing orders and notifications. Although this is a proof of concept, the principles and components can be expanded to suit production environments.


