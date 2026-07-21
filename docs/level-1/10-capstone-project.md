# 10 · Capstone — Serverless Web App

Time to combine everything from this level into one real, working
application: a static frontend hosted on S3, calling a Lambda function
through API Gateway, backed by a DynamoDB table — a simple "notes" app
where visitors can add a short note and see everyone else's notes. Every
piece of this has appeared in an earlier module individually; the capstone
is about wiring them together as one system, described as a single
CloudFormation stack, and — just as importantly — **tearing it all down
cleanly** afterward so nothing keeps billing you.

!!! warning "This module provisions real, billable AWS resources"
    Everything here fits comfortably in the AWS Free Tier for a training
    exercise (DynamoDB on-demand pricing, Lambda's 1M free requests/month, S3
    storage measured in KB). But free tier is not "impossible to be
    charged" — it's "free below a usage threshold, and only for 12 months on
    some resources." Follow the **Teardown** section at the end of this
    module every time you finish working on this project, not just once at
    the very end of the course.

## Architecture

```
Browser ──GET──▶ S3 static website (index.html + app.js)
   │
   └──POST/GET /notes──▶ API Gateway (HTTP API)
                              │
                              ▼
                        Lambda function
                              │
                              ▼
                        DynamoDB table (Notes)
```

- **S3** serves the static HTML/JS frontend (Module 4).
- **API Gateway** exposes `/notes` as a public HTTP endpoint (Module 7).
- **Lambda** contains the app logic: read all notes, or add one (Module 7).
- **DynamoDB** is a fully managed NoSQL database, storing each note as an
  item — new in this module, but conceptually parallel to RDS (Module 6):
  managed, no server to patch, but key-value/document shaped rather than
  relational.

## The DynamoDB table

```bash
aws dynamodb create-table \
  --table-name training-notes \
  --attribute-definitions AttributeName=noteId,AttributeType=S \
  --key-schema AttributeName=noteId,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST
```

`PAY_PER_REQUEST` (on-demand) billing charges only for reads/writes you
actually make — no capacity to provision or forget about, which is the
right default for a low-traffic training app.

## The Lambda function

```python
# handler.py
import json, os, uuid, time, boto3

dynamodb = boto3.resource("dynamodb")
table = dynamodb.Table(os.environ["TABLE_NAME"])

def handler(event, context):
    method = event["requestContext"]["http"]["method"]

    if method == "GET":
        items = table.scan().get("Items", [])
        items.sort(key=lambda i: i["createdAt"], reverse=True)
        return _response(200, items)

    if method == "POST":
        body = json.loads(event.get("body") or "{}")
        text = (body.get("text") or "").strip()
        if not text:
            return _response(400, {"error": "text is required"})
        note = {
            "noteId": str(uuid.uuid4()),
            "text": text[:280],
            "createdAt": int(time.time()),
        }
        table.put_item(Item=note)
        return _response(201, note)

    return _response(405, {"error": "method not allowed"})


def _response(status, body):
    return {
        "statusCode": status,
        "headers": {
            "Content-Type": "application/json",
            "Access-Control-Allow-Origin": "*",
        },
        "body": json.dumps(body),
    }
```

Package and prepare the execution role (least-privilege: only this table,
only the actions the function actually uses):

```bash
zip function.zip handler.py

cat > lambda-trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "lambda.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }]
}
EOF

aws iam create-role \
  --role-name notes-app-lambda-role \
  --assume-role-policy-document file://lambda-trust-policy.json

aws iam attach-role-policy \
  --role-name notes-app-lambda-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

cat > dynamodb-notes-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["dynamodb:PutItem", "dynamodb:Scan"],
    "Resource": "arn:aws:dynamodb:us-east-1:123456789012:table/training-notes"
  }]
}
EOF

aws iam put-role-policy \
  --role-name notes-app-lambda-role \
  --policy-name NotesTableAccess \
  --policy-document file://dynamodb-notes-policy.json
```

## Deploy the function and wire up API Gateway

```bash
aws lambda create-function \
  --function-name notes-app \
  --runtime python3.12 \
  --role arn:aws:iam::123456789012:role/notes-app-lambda-role \
  --handler handler.handler \
  --zip-file fileb://function.zip \
  --timeout 10 --memory-size 128 \
  --environment "Variables={TABLE_NAME=training-notes}"

aws apigatewayv2 create-api \
  --name notes-app-api \
  --protocol-type HTTP \
  --target arn:aws:lambda:us-east-1:123456789012:function:notes-app
# ApiEndpoint: https://xyz123abc.execute-api.us-east-1.amazonaws.com

aws lambda add-permission \
  --function-name notes-app \
  --statement-id apigw-invoke \
  --action lambda:InvokeFunction \
  --principal apigateway.amazonaws.com \
  --source-arn "arn:aws:execute-api:us-east-1:123456789012:xyz123abc/*/*/*"
```

Test the API directly before wiring up the frontend:

```bash
curl -X POST https://xyz123abc.execute-api.us-east-1.amazonaws.com/notes \
  -H "Content-Type: application/json" -d '{"text": "first note!"}'

curl https://xyz123abc.execute-api.us-east-1.amazonaws.com/notes
```

## The frontend

```html
<!-- index.html -->
<!DOCTYPE html>
<html>
<head><title>Training Notes</title></head>
<body>
  <h1>Notes</h1>
  <input id="text" placeholder="Write a note..." maxlength="280">
  <button onclick="addNote()">Add</button>
  <ul id="notes"></ul>
  <script>
    const API = "https://xyz123abc.execute-api.us-east-1.amazonaws.com/notes";

    async function loadNotes() {
      const res = await fetch(API);
      const notes = await res.json();
      document.getElementById("notes").innerHTML =
        notes.map(n => `<li>${n.text}</li>`).join("");
    }

    async function addNote() {
      const input = document.getElementById("text");
      await fetch(API, {
        method: "POST",
        headers: {"Content-Type": "application/json"},
        body: JSON.stringify({text: input.value}),
      });
      input.value = "";
      loadNotes();
    }

    loadNotes();
  </script>
</body>
</html>
```

Deploy it to a public S3 static website bucket, exactly as in Module 4:

```bash
aws s3 mb s3://yourname-notes-app-2026 --region us-east-1

aws s3api put-public-access-block \
  --bucket yourname-notes-app-2026 \
  --public-access-block-configuration \
  BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=false,RestrictPublicBuckets=false

cat > bucket-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "PublicReadForWebsite",
    "Effect": "Allow",
    "Principal": "*",
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::yourname-notes-app-2026/*"
  }]
}
EOF
aws s3api put-bucket-policy --bucket yourname-notes-app-2026 --policy file://bucket-policy.json

aws s3api put-bucket-website \
  --bucket yourname-notes-app-2026 \
  --website-configuration '{"IndexDocument": {"Suffix": "index.html"}}'

aws s3 cp index.html s3://yourname-notes-app-2026/
```

Visit `http://yourname-notes-app-2026.s3-website-us-east-1.amazonaws.com`,
add a note, refresh, and see it persisted — the full loop, entry-frontend to
serverless-backend to managed-database, with no server you provisioned or
patched anywhere in the stack.

## Doing it as one CloudFormation template (recommended)

Rebuilding the above as a single template — following Module 9's pattern —
means the whole app (minus the frontend's own HTML content, which you still
upload separately with `aws s3 cp`) can be created and destroyed with one
command each:

```yaml
# notes-app-stack.yaml
AWSTemplateFormatVersion: "2010-09-09"
Parameters:
  BucketName:
    Type: String
Resources:
  NotesTable:
    Type: AWS::DynamoDB::Table
    Properties:
      TableName: training-notes
      BillingMode: PAY_PER_REQUEST
      AttributeDefinitions:
        - AttributeName: noteId
          AttributeType: S
      KeySchema:
        - AttributeName: noteId
          KeyType: HASH

  LambdaRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Principal: {Service: lambda.amazonaws.com}
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
      Policies:
        - PolicyName: NotesTableAccess
          PolicyDocument:
            Version: "2012-10-17"
            Statement:
              - Effect: Allow
                Action: ["dynamodb:PutItem", "dynamodb:Scan"]
                Resource: !GetAtt NotesTable.Arn

  WebsiteBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Ref BucketName
      WebsiteConfiguration:
        IndexDocument: index.html
      PublicAccessBlockConfiguration:
        BlockPublicAcls: false
        IgnorePublicAcls: false
        BlockPublicPolicy: false
        RestrictPublicBuckets: false

Outputs:
  TableName:
    Value: !Ref NotesTable
  LambdaRoleArn:
    Value: !GetAtt LambdaRole.Arn
  WebsiteURL:
    Value: !GetAtt WebsiteBucket.WebsiteURL
```

(Lambda function code and API Gateway resources are omitted here for
brevity — a full production version would add `AWS::Lambda::Function`,
`AWS::ApiGatewayV2::Api`, `AWS::ApiGatewayV2::Integration`, and
`AWS::ApiGatewayV2::Route` resources, wiring `!GetAtt` references between
them the same way `LambdaRole` links to `NotesTable` above.)

## Teardown — do this every time

This is the most important section in this module. Delete resources in
reverse order of creation so nothing is left orphaned and billing:

```bash
# 1. Empty and delete the frontend bucket
aws s3 rm s3://yourname-notes-app-2026 --recursive
aws s3 rb s3://yourname-notes-app-2026

# 2. Remove the API Gateway invoke permission and the API itself
aws apigatewayv2 delete-api --api-id xyz123abc

# 3. Delete the Lambda function
aws lambda delete-function --function-name notes-app

# 4. Detach and delete the IAM role (inline policy first, then the role)
aws iam delete-role-policy --role-name notes-app-lambda-role --policy-name NotesTableAccess
aws iam detach-role-policy --role-name notes-app-lambda-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
aws iam delete-role --role-name notes-app-lambda-role

# 5. Delete the DynamoDB table
aws dynamodb delete-table --table-name training-notes

# 6. If you also worked through earlier modules and still have them running:
#    terminate any EC2 instances, delete the RDS instance, and delete any
#    CloudFormation stacks from Module 9.
aws ec2 describe-instances --filters "Name=instance-state-name,Values=running" \
  --query "Reservations[].Instances[].InstanceId" --output table
aws rds describe-db-instances --query "DBInstances[].DBInstanceIdentifier" --output table
aws cloudformation list-stacks --stack-status-filter CREATE_COMPLETE UPDATE_COMPLETE \
  --query "StackSummaries[].StackName" --output table
```

If you built the app via the CloudFormation template instead, teardown is
one command (plus emptying the bucket first, since CloudFormation won't
delete a non-empty bucket):

```bash
aws s3 rm s3://yourname-notes-app-2026 --recursive
aws cloudformation delete-stack --stack-name notes-app-stack
aws cloudformation wait stack-delete-complete --stack-name notes-app-stack
```

!!! warning "Make a final sweep across every service you touched this level"
    It's easy to forget a resource from an earlier module (an EC2 instance
    left running, an RDS database from Module 6). Run the `describe-*`
    commands above periodically, or check the **Billing Dashboard** in the
    AWS Console (Billing → Bills) to see exactly what's accruing charges
    right now, service by service.

## Cheat sheet

| Command | Purpose |
|---|---|
| `aws dynamodb create-table --billing-mode PAY_PER_REQUEST` | Create a NoSQL table with on-demand billing (no capacity to manage). |
| `aws dynamodb delete-table` | Delete a DynamoDB table. |
| `aws lambda delete-function` | Delete a Lambda function. |
| `aws apigatewayv2 delete-api` | Delete an HTTP API. |
| `aws iam delete-role-policy` / `detach-role-policy` / `delete-role` | Fully remove an IAM role (inline + managed policies first). |
| `aws s3 rm s3://BUCKET --recursive` then `aws s3 rb s3://BUCKET` | Empty and delete an S3 bucket. |
| `aws cloudformation delete-stack` | Delete every resource a stack owns, in one command. |
| Billing → Bills (Console) | See exactly what's currently accruing charges. |

## Exercise

1. Build the notes app following the steps above: DynamoDB table, Lambda
   function with a scoped-down execution role, HTTP API, and an S3-hosted
   frontend.
2. Add three notes through the browser UI and confirm all three show up on
   refresh (proving the full round trip: browser → API Gateway → Lambda →
   DynamoDB → back to browser).
3. Rebuild just the table + role portion as a CloudFormation template (the
   skeleton above), deploy it alongside your manually-created Lambda/API
   Gateway resources, and confirm `describe-stacks` shows it `CREATE_COMPLETE`.
4. Work through the full **Teardown** section, in order, for everything you
   created in this module.
5. Run the three `describe-*` sweep commands at the end of the Teardown
   section and confirm they return empty (or only resources from other
   courses/projects you intend to keep) — this is your habit-forming
   checkpoint before moving on to Level 2.
