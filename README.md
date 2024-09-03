# Proof of Concept: Serverless Web Backend on AWS

This project demonstrates how to build a proof of concept for a serverless solution on AWS. The solution is designed for a customer selling cleaning supplies, requiring a scalable architecture that handles spikes in demand while ensuring decoupled application components.

## Architecture Overview

The architecture consists of the following components:

1. **REST API (API Gateway)**: Receives incoming requests and places a database entry in the Amazon SQS queue.
2. **Amazon SQS**: Stores the data temporarily before triggering the first Lambda function.
3. **AWS Lambda (Function 1)**: Inserts the entry into a DynamoDB table.
4. **DynamoDB Streams**: Captures the new entry and invokes a second Lambda function.
5. **AWS Lambda (Function 2)**: Passes the database entry to Amazon SNS.
6. **Amazon SNS**: Sends a notification to a specified email address.

## Table of Contents

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
