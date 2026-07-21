# 07 · Lambda & Serverless

Lambda runs your code in response to an event — an HTTP request, a file
landing in S3, a schedule — without you provisioning or managing any server.
You pay only for the milliseconds your code actually executes, and it scales
from zero to thousands of concurrent invocations automatically. This module
covers writing and deploying a Lambda function, giving it an IAM execution
role, triggering it via API Gateway, and understanding the "serverless"
model well enough to build the capstone project.

## Core concepts

| Concept | What it is |
|---|---|
| **Function** | Your code (a handler) plus its runtime, memory, and timeout configuration. |
| **Execution role** | An IAM role Lambda assumes to run your code — this, not the caller's identity, determines what your function can access. |
| **Trigger** | The event source that invokes the function (API Gateway, S3, EventBridge, SQS, etc.). |
| **Cold start** | The latency of provisioning a fresh execution environment for the first invocation (or after scaling up). |
| **Concurrency** | The number of simultaneous invocations Lambda runs at once, scaled automatically per request. |

## Write a function

```python
# handler.py
import json

def handler(event, context):
    name = event.get("queryStringParameters", {}).get("name", "world") \
        if event.get("queryStringParameters") else "world"
    return {
        "statusCode": 200,
        "headers": {"Content-Type": "application/json"},
        "body": json.dumps({"message": f"Hello, {name}!"})
    }
```

Package it into a zip (Lambda deploys from a zip or container image):

```bash
zip function.zip handler.py
```

## Execution role: least privilege for the function itself

The function needs an IAM role it assumes at runtime — separate from the
identity of whoever *invokes* it:

```bash
cat > lambda-trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "lambda.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

aws iam create-role \
  --role-name training-lambda-role \
  --assume-role-policy-document file://lambda-trust-policy.json

# The bare minimum every Lambda function needs: permission to write logs
aws iam attach-role-policy \
  --role-name training-lambda-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
```

Add more permissions only as your function actually needs them (e.g.
`AmazonDynamoDBFullAccess` if it reads/writes DynamoDB) — the same
least-privilege principle from Module 2 applies here.

## Create the function

```bash
aws lambda create-function \
  --function-name training-hello \
  --runtime python3.12 \
  --role arn:aws:iam::123456789012:role/training-lambda-role \
  --handler handler.handler \
  --zip-file fileb://function.zip \
  --timeout 10 \
  --memory-size 128
```

Invoke it directly to test, without any trigger wired up yet:

```bash
aws lambda invoke \
  --function-name training-hello \
  --payload '{"queryStringParameters": {"name": "AWS"}}' \
  --cli-binary-format raw-in-base64-out \
  response.json

cat response.json
# {"statusCode": 200, "headers": {...}, "body": "{\"message\": \"Hello, AWS!\"}"}
```

## Updating code

```bash
# After editing handler.py and re-zipping
zip function.zip handler.py
aws lambda update-function-code \
  --function-name training-hello \
  --zip-file fileb://function.zip
```

## Triggering via API Gateway

To make the function reachable over plain HTTP, front it with an **HTTP
API** (API Gateway's simpler, cheaper API type — as opposed to the older,
more feature-rich REST API type):

```bash
# Create the HTTP API with Lambda as its target
aws apigatewayv2 create-api \
  --name training-api \
  --protocol-type HTTP \
  --target arn:aws:lambda:us-east-1:123456789012:function:training-hello
# ApiId: abc123xyz, ApiEndpoint: https://abc123xyz.execute-api.us-east-1.amazonaws.com

# API Gateway needs explicit permission to invoke your function
aws lambda add-permission \
  --function-name training-hello \
  --statement-id apigw-invoke \
  --action lambda:InvokeFunction \
  --principal apigateway.amazonaws.com \
  --source-arn "arn:aws:execute-api:us-east-1:123456789012:abc123xyz/*/*/*"
```

```bash
curl "https://abc123xyz.execute-api.us-east-1.amazonaws.com/?name=Reader"
# {"message": "Hello, Reader!"}
```

`aws lambda add-permission` is a resource-based policy on the *function*
itself (separate from the execution role) — it answers "who is allowed to
invoke this function," where the execution role answers "what can this
function do once it's running."

## Environment variables and configuration

```bash
aws lambda update-function-configuration \
  --function-name training-hello \
  --environment "Variables={STAGE=training,LOG_LEVEL=info}"
```

```python
import os
stage = os.environ["STAGE"]  # "training"
```

## Why "serverless" doesn't mean "no servers"

Servers still run your code — you simply never provision, patch, or manage
them. The tradeoffs versus EC2:

| | Lambda | EC2 |
|---|---|---|
| Billing | Per millisecond of actual execution | Per hour/second the instance is running, whether busy or idle |
| Scaling | Automatic, per-request, to zero | Manual or via Auto Scaling groups (Level 2) |
| Max run time | 15 minutes per invocation | Unlimited |
| Best for | Bursty, event-driven, short-lived work | Long-running processes, full OS control, steady load |

## Cheat sheet

| Command | Purpose |
|---|---|
| `aws iam create-role` + `attach-role-policy` | Create the function's execution role. |
| `aws lambda create-function --runtime R --role ARN --handler H --zip-file fileb://F` | Deploy a new function. |
| `aws lambda update-function-code --zip-file fileb://F` | Push new code to an existing function. |
| `aws lambda update-function-configuration --environment "Variables={K=V}"` | Set environment variables / memory / timeout. |
| `aws lambda invoke --function-name N --payload JSON response.json` | Test-invoke directly. |
| `aws apigatewayv2 create-api --protocol-type HTTP --target LAMBDA_ARN` | Create an HTTP API in front of the function. |
| `aws lambda add-permission --principal apigateway.amazonaws.com` | Let API Gateway invoke the function. |
| `aws lambda delete-function --function-name N` | Delete the function. |

## Exercise

1. Write a Lambda function in Python that accepts a JSON body
   `{"a": 1, "b": 2}` and returns their sum as JSON.
2. Create an execution role with just `AWSLambdaBasicExecutionRole`, deploy
   the function, and confirm it works with `aws lambda invoke`.
3. Front it with an HTTP API via `apigatewayv2 create-api`, grant the
   invoke permission, and call it with `curl -X POST` and a JSON body.
4. Check CloudWatch Logs (next module covers this properly, but try
   `aws logs describe-log-groups --log-group-name-prefix /aws/lambda/`) to
   find the log group Lambda created automatically for your function.
5. Delete the function and the API when done — Lambda's free tier is
   generous (1M free requests/month), but it's good practice to clean up
   training resources.
