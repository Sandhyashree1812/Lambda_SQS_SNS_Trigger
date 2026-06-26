# Lambda_SQS_SNS_Trigger
Step 1 — IAM: Create the Lambda Execution Role
Lambda needs an IAM role to:

Be assumed by the Lambda service
Read messages from SQS (and delete them after processing)
Publish messages to SNS
Write logs to CloudWatch
1.1 Open IAM
In the AWS Console, search for IAM and open it.
In the left sidebar click Roles → Create role.
1.2 Configure the Trust Policy
Field	Value
Trusted entity type	AWS service
Use case	Lambda
Click Next.

1.3 Attach Permissions
Search for and attach these two AWS managed policies:

Policy name	Why
AWSLambdaSQSQueueExecutionRole	Grants sqs:ReceiveMessage, sqs:DeleteMessage, sqs:GetQueueAttributes — the minimum for SQS triggers
AmazonSNSFullAccess	Grants sns:Publish so Lambda can post notifications
Least-privilege note: AmazonSNSFullAccess is used here for simplicity. In production, create a custom policy scoped to only sns:Publish on your specific topic ARN.

Click Next.

1.4 Name the Role
Field	Value
Role name	OrderProcessorLambdaRole
Description	Execution role for OrderProcessor Lambda — SQS trigger + SNS publish
Click Create role.

1.5 Note the Role ARN
Open the role you just created.
Copy the ARN (looks like arn:aws:iam::123456789012:role/OrderProcessorLambdaRole).
You will paste this ARN when creating the Lambda function in Step 4.

CloudWatch Logs
AWSLambdaSQSQueueExecutionRole already includes the logs:CreateLogGroup, logs:CreateLogStream, and logs:PutLogEvents permissions that Lambda needs to write to CloudWatch. No extra policy is needed.

Step 2 — SQS: Create OrderQueue and Dead-Letter Queue
You will create two queues:

OrderDLQ — receives messages that Lambda fails to process after several attempts
OrderQueue — the main inbound queue; Lambda is triggered by messages here
Create the DLQ first because the main queue references it.

2.1 Create the Dead-Letter Queue (OrderDLQ)
In the AWS Console, search for SQS and open it.
Click Create queue.
Field	Value
Type	Standard
Name	OrderDLQ
Leave all other settings at their defaults and click Create queue.

Copy the Queue ARN for OrderDLQ — you will need it in the next section.
2.2 Create the Main Queue (OrderQueue)
Click Create queue again.
Field	Value
Type	Standard
Name	OrderQueue
Scroll down to Dead-letter queue and expand it.
Field	Value
Dead-letter queue	Enabled
Choose queue	OrderDLQ (paste the ARN you copied)
Maximum receives	3
Maximum receives = 3 means: if the same message is received and not deleted 3 times (Lambda threw an exception each time), SQS moves it to the DLQ automatically.

Leave all other settings at defaults. Click Create queue.

Copy the Queue URL and Queue ARN for OrderQueue — you will need these in later steps.

What You Created
OrderQueue (Standard)
  └── Dead-letter queue → OrderDLQ (maxReceiveCount: 3)
When Lambda fails to process a message 3 times, SQS automatically moves it to OrderDLQ so it isn't lost and can be investigated.

tep 3 — SNS: Create Topics and Subscribe
You will create two SNS topics:

Topic	Purpose
OrderNotifications	Business events — published on successful processing; consumed by fulfillment, inventory, etc.
OrderAlerts	Operational alerts — published on every success AND failure; consumed by operators and monitoring tools
Separating concerns means a failed message alert doesn't land in the same topic as a downstream fulfillment event.

3.1 Create OrderNotifications Topic
In the AWS Console, search for SNS and open it.
In the left sidebar click Topics → Create topic.
Field	Value
Type	Standard
Name	OrderNotifications
Click Create topic.

Copy the Topic ARN — you will set this as SNS_TOPIC_ARN in the Lambda environment variables.
3.2 Create OrderAlerts Topic
Click Create topic again.
Field	Value
Type	Standard
Name	OrderAlerts
Click Create topic.

Copy the Topic ARN — you will set this as ALERT_SNS_TOPIC_ARN in the Lambda environment variables.
3.3 Subscribe Your Email to OrderAlerts
This gives you real-time visibility into every success and failure the Lambda processes.

With OrderAlerts open, click Create subscription.
Field	Value
Protocol	Email
Endpoint	your email address
Click Create subscription.

Check your inbox for a confirmation email from AWS and click Confirm subscription.
You will not receive any alerts until you confirm. Check your spam folder if it does not arrive within a minute.

3.4 (Optional) Filter Alerts by Status
If you only want to receive failure emails — not one for every successful order — add a subscription filter policy:

SNS → OrderAlerts → Subscriptions → click your email subscription → Edit.
Under Subscription filter policy paste:
{
  "status": ["FAILED"]
}
Click Save changes. Now your email only receives FAILED alerts; SUCCESS alerts are still published but filtered out for this subscriber.

The status attribute is set by the Lambda function in MessageAttributes — SUCCESS or FAILED — so SNS can route them differently per subscriber.

3.5 Subscribe Your Email to OrderNotifications
This lets you see the business notification that downstream services would receive.

Open OrderNotifications → Create subscription.
Field	Value
Protocol	Email
Endpoint	your email address
Click Create subscription and confirm from your inbox.

3.6 Create a "ProcessedOrders" SQS Queue
This queue simulates a downstream service (e.g., a fulfillment system) consuming processed-order events.

Go to SQS → Create queue.
Field	Value
Type	Standard
Name	ProcessedOrders
Click Create queue.

Copy the Queue ARN for ProcessedOrders.
3.7 Subscribe ProcessedOrders Queue to OrderNotifications
Back in SNS → Topics → OrderNotifications → Create subscription.
Field	Value
Protocol	Amazon SQS
Endpoint	ARN of ProcessedOrders
Click Create subscription.

3.8 Add SQS Access Policy for SNS Delivery
SNS must be allowed to send messages to the ProcessedOrders queue.

Go to SQS → ProcessedOrders → Access policy tab → Edit.
Replace the existing policy with the following (substitute your values):
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowSNSPublish",
      "Effect": "Allow",
      "Principal": {
        "Service": "sns.amazonaws.com"
      },
      "Action": "sqs:SendMessage",
      "Resource": "<ProcessedOrders-Queue-ARN>",
      "Condition": {
        "ArnEquals": {
          "aws:SourceArn": "<OrderNotifications-Topic-ARN>"
        }
      }
    }
  ]
}
Click Save.

What You Created
OrderNotifications (SNS Topic)  ← business events (success only)
  ├── Email subscription  → your inbox
  └── SQS subscription   → ProcessedOrders queue

OrderAlerts (SNS Topic)          ← ops visibility (success + failure)
  └── Email subscription  → your inbox
        (optional filter: status = FAILED)

        Step 4 — Lambda: Create and Deploy the Function
You will create the Lambda function, upload the code, and set the SNS topic ARN as an environment variable.

4.1 Open Lambda
In the AWS Console, search for Lambda and open it.
Click Create function.
4.2 Basic Configuration
Field	Value
Author from scratch	selected
Function name	OrderProcessor
Runtime	Python 3.14
Architecture	x86_64
4.3 Attach the Execution Role
Under Permissions → Change default execution role:

Field	Value
Execution role	Use an existing role
Existing role	OrderProcessorLambdaRole (created in Step 1)
Click Create function.

4.4 Upload the Code
The Lambda code is saved as handler.py :

import json
import os
import boto3

sns_client = boto3.client("sns")

# Business events topic — downstream systems (fulfillment, inventory, etc.)
SNS_TOPIC_ARN = os.environ["SNS_TOPIC_ARN"]

# Operational alerts topic — success/failure visibility for operators and monitoring.
# Optional: if not set, alerts are skipped and only CloudWatch logs are written.
ALERT_SNS_TOPIC_ARN = os.environ.get("ALERT_SNS_TOPIC_ARN")


def _publish_alert(status: str, subject: str, payload: dict) -> None:
    """Publish a success or failure alert to the ops alert topic.
    status must be 'SUCCESS' or 'FAILED' — used as an SNS filter attribute.
    """
    if not ALERT_SNS_TOPIC_ARN:
        return
    sns_client.publish(
        TopicArn=ALERT_SNS_TOPIC_ARN,
        Subject=subject,
        Message=json.dumps(payload),
        MessageAttributes={
            "status": {
                "DataType": "String",
                "StringValue": status,
            }
        },
    )


def lambda_handler(event, context):
    """
    Triggered by SQS. Processes each record and publishes results to SNS.

    On success: publishes the processed order to OrderNotifications (business topic)
                and an alert to OrderAlerts (ops topic) with status=SUCCESS.
    On failure: returns the message ID in batchItemFailures so SQS retries it,
                and publishes an alert to OrderAlerts with status=FAILED.
    """
    processed = []
    failed = []

    for record in event["Records"]:
        message_id = record["messageId"]
        try:
            body = json.loads(record["body"])
            order_id = body.get("orderId", "UNKNOWN")
            customer = body.get("customer", "Unknown Customer")
            amount = body.get("amount", 0)

            notification = {
                "orderId": order_id,
                "customer": customer,
                "amount": amount,
                "status": "PROCESSED",
                "message": f"Order {order_id} for {customer} (${amount}) has been processed.",
            }

            # Publish to business downstream topic
            sns_client.publish(
                TopicArn=SNS_TOPIC_ARN,
                Subject=f"Order Processed: {order_id}",
                Message=json.dumps(notification),
                MessageAttributes={
                    "eventType": {
                        "DataType": "String",
                        "StringValue": "ORDER_PROCESSED",
                    }
                },
            )

            # Publish success alert to ops topic
            _publish_alert(
                status="SUCCESS",
                subject=f"SUCCESS: Order {order_id} processed",
                payload={
                    "status": "SUCCESS",
                    "orderId": order_id,
                    "customer": customer,
                    "amount": amount,
                    "sqsMessageId": message_id,
                },
            )

            print(f"Published SNS notification for order {order_id}")
            processed.append(message_id)

        except Exception as e:
            print(f"Failed to process message {message_id}: {e}")

            # Publish failure alert to ops topic before returning batchItemFailures
            _publish_alert(
                status="FAILED",
                subject=f"FAILED: Message processing error ({message_id})",
                payload={
                    "status": "FAILED",
                    "sqsMessageId": message_id,
                    "error": str(e),
                    # Include raw body so operators can inspect what was received
                    "rawBody": record.get("body", ""),
                },
            )

            # Returning a batchItemFailures response tells SQS to retry only
            # the failed messages rather than the entire batch.
            failed.append({"itemIdentifier": message_id})

    print(f"Processed: {len(processed)}, Failed: {len(failed)}")

    if failed:
        return {"batchItemFailures": failed}

    return {"statusCode": 200, "body": f"Processed {len(processed)} message(s)"}







=======================================

Option A — Inline editor (quickest for this project):

On the function page, scroll to the Code source section.
Click on lambda_function.py in the file tree and delete its contents.
Paste the entire contents of handler.py into the editor.
Click Deploy.
Option B — ZIP upload:

cd lambda-sqs-sns-trigger/lambda
zip function.zip handler.py
On the function page → Upload from → .zip file → select function.zip.
Click Save.
Note: If you use Option A, the handler is lambda_function.lambda_handler by default. You need to change it to match the file name you pasted into. Either rename the file to lambda_function.py or update the handler setting (next section).

4.5 Set the Handler
If you pasted the code into lambda_function.py (the default file), no change is needed.

If you uploaded handler.py:

Click Configuration tab → General configuration → Edit.
Set Handler to handler.lambda_handler.
Click Save.
4.6 Set Environment Variables
Click Configuration tab → Environment variables → Edit.
Add both variables using Add environment variable:
Key	Value	Required
SNS_TOPIC_ARN	ARN of OrderNotifications (copied in Step 3.1)	Yes
ALERT_SNS_TOPIC_ARN	ARN of OrderAlerts (copied in Step 3.2)	Optional — if omitted, alerts are skipped and only CloudWatch logs are written
Click Save.

Why two topics? SNS_TOPIC_ARN carries business events consumed by downstream services. ALERT_SNS_TOPIC_ARN carries operational signals (success + failure) for operators and monitoring — keeping these separate means a processing failure doesn't pollute your fulfillment topic.

4.7 (Optional) Tune Memory and Timeout
The defaults (128 MB memory, 3-second timeout) are fine for this project. For real workloads, adjust under Configuration → General configuration.

What You Deployed
The function:

Receives an SQS batch of records
Parses each record as a JSON order
On success: publishes the processed order to OrderNotifications; publishes a SUCCESS alert to OrderAlerts
On failure: publishes a FAILED alert to OrderAlerts with the error message and raw SQS body; returns the message ID in batchItemFailures so SQS retries only that record

Step 5 — Event Source Mapping: Connect SQS to Lambda
The event source mapping tells Lambda to poll OrderQueue and invoke OrderProcessor automatically whenever messages arrive. You do not write any polling code — AWS manages it.

5.1 Add the Trigger
Open the OrderProcessor Lambda function.
Click + Add trigger (on the Function overview diagram).
Field	Value
Trigger configuration	SQS
SQS queue	OrderQueue
Batch size	5
Batch window	0 seconds
Report batch item failures	Enabled
Batch size = 5: Lambda receives up to 5 messages in a single invocation. For this project, 1–5 is fine.

Report batch item failures: Must be enabled so Lambda's batchItemFailures response is respected. Without it, a single failed message causes the entire batch to be retried.

Click Add.

5.2 Verify the Trigger Appears
After a few seconds you should see SQS listed as a trigger in the Function overview. The state should be Enabled.

How It Works Internally
SQS polls internally (AWS-managed)
    │
    │  up to 5 messages at a time
    ▼
Lambda invocation
    │
    ├── success  → SQS deletes those messages automatically
    └── failure  → those messageIds returned in batchItemFailures
                       └── SQS retries them (up to maxReceiveCount=3)
                                └── then moves to OrderDLQ
Lambda's event source mapping handles all of this. You only write the handler logic.

5.3 IAM Check
The AWSLambdaSQSQueueExecutionRole policy attached in Step 1 already grants:

sqs:ReceiveMessage
sqs:DeleteMessage
sqs:GetQueueAttributes
No additional SQS resource policy is needed for the Lambda trigger.

Step 6 — Testing: Send a Message and Verify the Pipeline
You will send a test order to OrderQueue and verify that Lambda processes it and SNS delivers notifications.

6.1 Send a Test Message via the Console
In the AWS Console, go to SQS → OrderQueue.
Click Send and receive messages → Send message.
Paste this JSON into the Message body field:
{
  "orderId": "ORD-001",
  "customer": "Jane Doe",
  "amount": 49.99,
  "items": ["Widget A", "Widget B"]
}
Click Send message.
6.2 Send a Test Message via AWS CLI
If you have the AWS CLI configured:

aws sqs send-message \
  --queue-url <OrderQueue-URL> \
  --message-body '{"orderId":"ORD-002","customer":"John Smith","amount":129.00,"items":["Gadget X"]}'
Replace <OrderQueue-URL> with the URL you copied in Step 2.

6.3 Verify Lambda Was Invoked
Open Lambda → OrderProcessor → Monitor tab → View CloudWatch logs.
Click the most recent log stream.
You should see log lines similar to:

Published SNS notification for order ORD-001
Processed: 1, Failed: 0
6.4 Verify Email Notification
Check the inbox you subscribed in Step 3. You should receive an email with:

Subject: Order Processed: ORD-001
Body: JSON with status: "PROCESSED" and order details
It may take up to 60 seconds to arrive.

6.5 Verify ProcessedOrders Queue
Go to SQS → ProcessedOrders.
Click Send and receive messages → Poll for messages.
You should see a message. Click on it to inspect the body — it will contain the SNS envelope wrapping the notification JSON.
6.6 Test a Failure (DLQ Path)
Send a malformed message to verify the DLQ works:

aws sqs send-message \
  --queue-url <OrderQueue-URL> \
  --message-body 'this is not valid json'
Lambda will fail to json.loads this body. After 3 receive attempts (maxReceiveCount), SQS moves it to OrderDLQ.

Verify:

Wait ~2 minutes (SQS visibility timeout + retries).
Go to SQS → OrderDLQ → Send and receive messages → Poll for messages.
The malformed message should appear there.
6.7 Lambda Test Event (Console)
You can also test directly from the Lambda console without sending an SQS message:

Open OrderProcessor → Test tab → Create new event.
Select Template → search for sqs → choose SQS.
Replace the body field in the template with:
"{\"orderId\":\"ORD-TEST\",\"customer\":\"Test User\",\"amount\":9.99}"
Click Test. Check the Execution results and CloudWatch logs.

Step 7 — Cleanup: Delete All Resources
Delete resources in this order to avoid dependency errors.

7.1 Delete the Lambda Event Source Mapping
Open Lambda → OrderProcessor → Configuration → Triggers.
Select the SQS trigger → Delete.
7.2 Delete the Lambda Function
Go to Lambda → Functions.
Select OrderProcessor → Actions → Delete.
Type delete to confirm.
7.3 Delete the SNS Subscriptions and Topics
Go to SNS → Subscriptions.
Select all subscriptions for OrderNotifications (email + SQS) → Delete.
Select the email subscription for OrderAlerts → Delete.
Go to SNS → Topics.
Select OrderNotifications → Delete → type delete me to confirm.
Select OrderAlerts → Delete → type delete me to confirm.
7.4 Delete the SQS Queues
Go to SQS.
Select OrderQueue → Delete → confirm.
Select ProcessedOrders → Delete → confirm.
Select OrderDLQ → Delete → confirm.
7.5 Delete the IAM Role
Go to IAM → Roles.
Search for OrderProcessorLambdaRole.
Select it → Delete → type the role name to confirm.
7.6 Delete CloudWatch Log Group
Lambda automatically creates a log group. To delete it:

Go to CloudWatch → Log groups.
Find /aws/lambda/OrderProcessor.
Select it → Actions → Delete log group(s) → confirm.
Verification
After cleanup, verify:

Lambda console shows no OrderProcessor function
SQS console shows none of the three queues (OrderQueue, OrderDLQ, ProcessedOrders)
SNS console shows neither OrderNotifications nor OrderAlerts
IAM console shows no OrderProcessorLambdaRole
You will not be charged for any of the resources used in this project if they are deleted promptly.
