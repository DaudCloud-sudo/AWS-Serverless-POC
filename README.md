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

![image](https://github.com/user-attachments/assets/3cb0991f-2108-4c45-9b47-d88d667c12a2)

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

![image](https://github.com/user-attachments/assets/4b4ed5f3-902d-4154-b1e9-cc2c3aefcd77)

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

![image](https://github.com/user-attachments/assets/75354823-c014-4b6c-b7d4-c270943cc5ad)
![image](https://github.com/user-attachments/assets/a5742cf6-8e55-4582-86c4-75278c2769c5)

### Task 5: Enabling DynamoDB Streams

1. In the DynamoDB console, select the `orders` table.
2. Go to the Exports and streams tab.
3. Enable DynamoDB Streams with the New image view type.

![image](https://github.com/user-attachments/assets/f46af974-da99-4e78-883a-1b45822c801e)

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

![image](https://github.com/user-attachments/assets/808799ab-d02e-4912-9ebf-4d55329699c4)

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

![image](https://github.com/user-attachments/assets/ecbc0391-2775-4f87-8ded-958e6fb07334)

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

#### Step 7.4: Testing the POC-Lambda-2 Lambda function

1. On the **Test** tab, create a new event and for Event name, enter `POC-Lambda-Test-2`.
2. For Template-optional, enter DynamoDB and from the list, choose DynamoDB-Update.
3. The DynamoDB template appears in the Event JSON box.
4. Save your changes and choose **Test**.

After the Lambda function successfully runs, the “Execution result: succeeded” message should appear in the notification banner in the Test section. In a few minutes, an email message should arrive at the email address that you specified in the previous task. Confirm that you received the subscription email message. If needed, check both your inbox and spam folder.

![image](https://github.com/user-attachments/assets/ddfaf06a-ff73-4df5-8dab-51ed57ec70b7)

### Task 8: Creating an API with Amazon API Gateway

In this task, you will create a REST API in Amazon API Gateway. This API serves as a communication gateway between your application and the AWS services.

1. In the AWS Management Console, search for and open **API Gateway**.
2. On the **REST API** card with public authentication, choose **Build** and configure the following settings:
   - **Choose the protocol**: REST
   - **Create new API**: New API
   - **API name**: `POC-API`
   - **Endpoint Type**: Regional
   - Choose **Create API**.

3. On the **Actions** menu, choose **Create Method**.
4. Open the method menu by choosing the down arrow, and choose **POST**. Save your changes by choosing the check mark.
5. In the **POST - Setup** pane, configure the following settings:
   - **Integration type**: AWS Service
   - **AWS Region**: us-east-1
   - **AWS Service**: Simple Queue Service (SQS)
   - **AWS Subdomain**: Keep empty
   - **HTTP method**: POST
   - **Action Type**: Use path override
   - **Path override**: Enter your account ID followed by a slash (/) and the name of the `POC-Queue`
     - Note: If `POC-Queue` is the name of the SQS queue that you created, this entry might look similar to the following: `/<account ID>/POC-Queue`
   - **Execution role**: Paste the ARN of the `APIGateway-SQS` role
     - Note: For example, the ARN might look like the following: `arn:aws:iam::<account ID>:role/APIGateway-SQS`
   - **Content Handling**: Passthrough
   - Save your changes.

6. Choose the **Integration Request** card.
7. Scroll to the bottom of the page and expand **HTTP Headers**.
   - Choose **Add header**.
   - For **Name**, enter `Content-Type`.
   - For **Mapped from**, enter `'application/x-www-form-urlencoded'`.
   - Save your changes to the **HTTP Headers** section by choosing the check mark.

8. Expand **Mapping Templates** and for **Request body passthrough**, choose **Never**.
   - Choose **Add mapping template** and for **Content-Type**, enter `application/json`.
   - Save your changes by choosing the check mark.

9. For **Generate template**, do not choose a default template from the list. Instead, enter the following command: `Action=SendMessage&MessageBody=$input.body` in the box.
   - Choose **Save**.
  
![image](https://github.com/user-attachments/assets/518fc876-966e-4d4f-aa5c-7e070856fe27)

### Task 9: Testing the Architecture by Using API Gateway

In this task, you will use API Gateway to send mock data to Amazon SQS as a proof of concept for the serverless solution.

1. In the API Gateway console, return to the **POST - Method Execution** page and choose **Test**.
2. In the **Request Body** box, enter:
    ```json
    {
        "item": "latex gloves",
        "customerID": "12345"
    }
    ```
3. Choose **Test**.

![image](https://github.com/user-attachments/assets/b758dc69-747a-4955-b6bb-eeeac964f023)

4. Verification in DynamoDB Table.

![image](https://github.com/user-attachments/assets/9ff97686-3104-4182-8fa8-88ba47cbf064)

   - If you see the "Successfully completed execution" message with the 200 response in the logs on the right, you will receive an email notification with the new entry. If you don’t receive an email but the new item appears in the DynamoDB table, troubleshoot the exercise instructions starting from after you set up DynamoDB. Ensure that you deploy all resources in the `us-east-1` Region.

   - After API Gateway successfully processes the request pasted in the **Request Body** box, it places the request in the SQS queue. Amazon SQS, set up as a trigger in the first Lambda function, invokes the function call. The Lambda function code places the new entry into the DynamoDB table. DynamoDB Streams captures this change to the database and invokes the second AWS Lambda function. This function retrieves the new record from DynamoDB Streams and sends it to Amazon SNS. Amazon SNS then sends you an email notification.

### Task 10: Cleaning Up

In this task, you will delete the AWS resources that you created for this exercise.

1. **Delete the DynamoDB table:**
   - Open the DynamoDB console.
   - In the navigation pane, choose **Tables**.
   - Select the `orders` table.
   - Choose **Delete** and confirm your actions.

2. **Delete the Lambda functions:**
   - Open the Lambda console.
   - Select the Lambda functions that you created in this exercise: `POC-Lambda-1` and `POC-Lambda-2`.
   - Choose **Actions**, then **Delete**.
   - Confirm your actions and close the dialog box.

3. **Delete the SQS queue:**
   - Open the Amazon SQS console.
   - Select the queue that you created in this exercise.
   - Choose **Delete** and confirm your actions.

4. **Delete the SNS topic and subscriptions:**
   - Open the Amazon SNS console.
   - In the navigation pane, choose **Topics**.
   - Select `POC-Topic`.
   - Choose **Delete** and confirm your actions.
   - In the navigation pane, choose **Subscriptions**.
   - Select the subscription that you created in this exercise and choose **Delete**.
   - Confirm your actions.

5. **Delete the API that you created:**
   - Open the API Gateway console.
   - Select `POC-API`.
   - Choose **Actions**, then **Delete**.
   - Confirm your actions.

6. **Delete the IAM roles and policies:**
   - Open the IAM console.
   - In the navigation pane, choose **Roles**.
   - Delete the following roles and confirm your actions:
     - `APIGateway-SQS`
     - `Lambda-SQS-DynamoDB`
     - `Lambda-DynamoDBStreams-SNS`
   - In the navigation pane, choose **Policies**.
   - Delete the following custom policies and confirm your actions:
     - `Lambda-DynamoDBStreams-Read`
     - `Lambda-SNS-Publish`
     - `Lambda-Write-DynamoDB`
     - `Lambda-Read-SQS`

This serverless architecture provides a scalable solution for processing orders and notifications. Although this is a proof of concept, the principles and components can be expanded to suit production environments.


