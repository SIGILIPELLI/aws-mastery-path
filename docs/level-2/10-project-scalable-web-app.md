# 10 · Project — Scalable Web App

The Level 1 capstone combined S3, API Gateway, Lambda, and DynamoDB into
one serverless system. This project combines everything from Level 2 into
a **containerized, load-balanced, globally-distributed** system instead:
ECS/Fargate for compute, an ALB for traffic distribution and health
checks, DynamoDB for data, Route 53 for a real domain name, and
CloudFront in front of the static frontend. It's the same "notes app"
shape as the Level 1 capstone, rebuilt on the horizontally-scalable,
container-based stack this level covers, so you can compare the two
approaches directly.

## Architecture

```
Browser
  │
  ├──static assets (index.html, app.js)──▶ CloudFront ──▶ S3 (private, OAC)
  │
  └──api.training.example.com──▶ Route 53 (alias)──▶ ALB──▶ ECS/Fargate service
                                                                  │
                                                                  ▼
                                                          DynamoDB (Notes table)
```

- **Route 53** (module 3) resolves `api.training.example.com` to the ALB
  via an alias record, and `www.training.example.com` to CloudFront.
- **CloudFront** (module 8) serves the static frontend from a private S3
  bucket, cached at the edge.
- **ALB** (module 2) load-balances across ECS tasks and health-checks
  them before routing traffic.
- **ECS/Fargate** (module 1) runs the API as a containerized service, with
  an ASG-equivalent scaling policy via ECS Service Auto Scaling.
- **DynamoDB** (module 5) stores notes with a composite key, queryable by
  a GSI for "recent notes across all users."
- **Secrets Manager** (module 7) holds nothing sensitive here (DynamoDB
  access is via IAM task role, not credentials) — but is where you'd store
  any third-party API key this app needed.

## Step 1 — DynamoDB table

```bash
aws dynamodb create-table \
  --table-name scalable-app-notes \
  --attribute-definitions \
      AttributeName=userId,AttributeType=S \
      AttributeName=noteId,AttributeType=S \
      AttributeName=gsiPk,AttributeType=S \
      AttributeName=createdAt,AttributeType=S \
  --key-schema \
      AttributeName=userId,KeyType=HASH \
      AttributeName=noteId,KeyType=RANGE \
  --billing-mode PAY_PER_REQUEST \
  --global-secondary-indexes '[{
    "IndexName": "byRecency",
    "KeySchema": [
      {"AttributeName": "gsiPk", "KeyType": "HASH"},
      {"AttributeName": "createdAt", "KeyType": "RANGE"}
    ],
    "Projection": {"ProjectionType": "ALL"}
  }]'
```

`gsiPk` is set to a constant value (e.g. `"ALL"`) on every item — a
deliberate single-partition GSI pattern for a low-write "recent activity"
feed. It doesn't scale to huge write volume (module 5's hot-partition
warning applies directly here), but it's the simplest way to query "the
N most recent notes across all users," and is fine at this project's
scale.

## Step 2 — containerize the API and push to ECR

```bash
docker build -t scalable-app-api:latest .

aws ecr create-repository --repository-name scalable-app-api
aws ecr get-login-password --region us-east-1 \
  | docker login --username AWS --password-stdin 123456789012.dkr.ecr.us-east-1.amazonaws.com
docker tag scalable-app-api:latest 123456789012.dkr.ecr.us-east-1.amazonaws.com/scalable-app-api:latest
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/scalable-app-api:latest
```

The API container needs a **task role** (module 1) scoped to exactly
`dynamodb:GetItem`, `PutItem`, `Query` on `scalable-app-notes` and its
`byRecency` index — not broader DynamoDB access, and definitely not the
execution role, which stays scoped to ECR pull + CloudWatch Logs only.

## Step 3 — ALB and target group

```bash
aws elbv2 create-load-balancer \
  --name scalable-app-alb \
  --subnets subnet-0aaa1111 subnet-0bbb2222 \
  --security-groups sg-0123456789abcdef0 \
  --scheme internet-facing --type application

aws elbv2 create-target-group \
  --name scalable-app-tg \
  --protocol HTTP --port 8080 \
  --target-type ip \
  --vpc-id vpc-0123456789abcdef0 \
  --health-check-path /health

aws elbv2 create-listener \
  --load-balancer-arn arn:...:loadbalancer/app/scalable-app-alb/... \
  --protocol HTTP --port 80 \
  --default-actions Type=forward,TargetGroupArn=arn:...:targetgroup/scalable-app-tg/...
```

`--target-type ip` (rather than `instance`) is required for Fargate tasks —
they're registered with the target group by their ENI's private IP, since
there's no EC2 instance ID to register.

## Step 4 — ECS service wired to the ALB

```bash
aws ecs create-cluster --cluster-name scalable-app-cluster

aws ecs register-task-definition --cli-input-json file://task-def.json

aws ecs create-service \
  --cluster scalable-app-cluster \
  --service-name scalable-app-api-svc \
  --task-definition scalable-app-api:1 \
  --desired-count 2 \
  --launch-type FARGATE \
  --network-configuration '{
    "awsvpcConfiguration": {
      "subnets": ["subnet-0aaa1111", "subnet-0bbb2222"],
      "securityGroups": ["sg-0123456789abcdef0"],
      "assignPublicIp": "ENABLED"
    }
  }' \
  --load-balancers '[{
    "targetGroupArn": "arn:...:targetgroup/scalable-app-tg/...",
    "containerName": "api",
    "containerPort": 8080
  }]' \
  --health-check-grace-period-seconds 60
```

This is the ECS equivalent of module 2's `create-auto-scaling-group
--target-group-arns` — the service registers/deregisters tasks with the
target group automatically as it scales or replaces unhealthy tasks.

## Step 5 — ECS Service Auto Scaling

```bash
aws application-autoscaling register-scalable-target \
  --service-namespace ecs \
  --resource-id service/scalable-app-cluster/scalable-app-api-svc \
  --scalable-dimension ecs:service:DesiredCount \
  --min-capacity 2 --max-capacity 8

aws application-autoscaling put-scaling-policy \
  --service-namespace ecs \
  --resource-id service/scalable-app-cluster/scalable-app-api-svc \
  --scalable-dimension ecs:service:DesiredCount \
  --policy-name api-cpu-target-tracking \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{
    "PredefinedMetricSpecification": {"PredefinedMetricType": "ECSServiceAverageCPUUtilization"},
    "TargetValue": 50.0
  }'
```

ECS services scale through **Application Auto Scaling**, a shared scaling
API also used by DynamoDB provisioned capacity and other services —
conceptually identical to module 2's ASG target tracking, but the
"instance" being added or removed is a task, not an EC2 instance.

## Step 6 — CloudFront + S3 for the static frontend

```bash
aws s3 mb s3://scalable-app-frontend-2026
aws s3 cp ./frontend/ s3://scalable-app-frontend-2026/ --recursive

aws cloudfront create-origin-access-control --origin-access-control-config '{
  "Name": "scalable-app-oac", "OriginAccessControlOriginType": "s3",
  "SigningBehavior": "always", "SigningProtocol": "sigv4"
}'

aws cloudfront create-distribution --distribution-config file://frontend-distribution.json
```

The frontend's `app.js` calls `https://api.training.example.com/notes` —
a **different** hostname from the CloudFront domain serving the HTML/JS,
so the API needs CORS headers (`Access-Control-Allow-Origin`) configured
in the container's HTTP responses, or requests from the browser will be
blocked despite the API itself working fine via `curl`.

## Step 7 — Route 53 records

```bash
# api.training.example.com -> ALB
aws route53 change-resource-record-sets --hosted-zone-id /hostedzone/Z1D633PJN98FT9 \
  --change-batch '{"Changes":[{"Action":"UPSERT","ResourceRecordSet":{
    "Name":"api.training.example.com","Type":"A",
    "AliasTarget":{"HostedZoneId":"Z35SXDOTRQ7X7K","DNSName":"scalable-app-alb-123456789.us-east-1.elb.amazonaws.com","EvaluateTargetHealth":true}
  }}]}'

# www.training.example.com -> CloudFront
aws route53 change-resource-record-sets --hosted-zone-id /hostedzone/Z1D633PJN98FT9 \
  --change-batch '{"Changes":[{"Action":"UPSERT","ResourceRecordSet":{
    "Name":"www.training.example.com","Type":"A",
    "AliasTarget":{"HostedZoneId":"Z2FDTNDATAQYW2","DNSName":"d111111abcdef8.cloudfront.net","EvaluateTargetHealth":false}
  }}]}'
```

`Z2FDTNDATAQYW2` is CloudFront's fixed, global alias hosted zone ID (the
same constant in every account/region — unlike the ALB's zone ID, which
is region-specific and comes from `describe-load-balancers`).
`EvaluateTargetHealth` is `false` for the CloudFront alias because
CloudFront doesn't expose a Route 53-compatible health status the way an
ALB does.

## End-to-end verification

```bash
curl https://api.training.example.com/health
# {"status": "ok"}

curl -X POST https://api.training.example.com/notes \
  -H "Content-Type: application/json" \
  -d '{"userId": "demo", "text": "hello from the scalable stack"}'

curl https://api.training.example.com/notes?userId=demo

curl -I https://www.training.example.com/
# x-cache: Hit from cloudfront (on a second request)
```

## Failure-injection check

```bash
# Kill one task manually and confirm the service self-heals
aws ecs list-tasks --cluster scalable-app-cluster --service-name scalable-app-api-svc
aws ecs stop-task --cluster scalable-app-cluster --task <task-arn>

# Watch a replacement task appear and the ALB keep serving traffic throughout
aws ecs describe-services --cluster scalable-app-cluster --services scalable-app-api-svc \
  --query "services[0].[runningCount,desiredCount]"
```

If `curl https://api.training.example.com/notes` kept succeeding while
one task was down and being replaced, the ALB's health checking and ECS's
self-healing did exactly what modules 1 and 2 promised.

!!! warning "This project's moving pieces bill independently — verify each one is gone"
    Unlike the Level 1 capstone (mostly serverless, scales to zero cost at
    rest), this stack has an ALB and running Fargate tasks that bill
    continuously regardless of traffic. Tear down fully: ECS service
    (scale to 0, then delete) → cluster → target group → listener → load
    balancer → CloudFront distribution (disable, wait, delete) → Route 53
    records → DynamoDB table → S3 buckets → ECR repository.

## Cheat sheet

| Command | Purpose |
|---|---|
| `aws dynamodb create-table --global-secondary-indexes ...` | Data layer with a "recent items" query pattern. |
| `aws ecs create-service --load-balancers [...]` | Wire a Fargate service directly to an ALB target group. |
| `aws application-autoscaling register-scalable-target --service-namespace ecs` | Enable ECS service auto scaling. |
| `aws cloudfront create-distribution` | CDN in front of the static frontend. |
| `aws route53 change-resource-record-sets` | Point real domain names at the ALB and CloudFront. |
| `aws ecs stop-task` | Manually kill a task to test self-healing. |

## Stretch goals

- **HTTPS on the ALB**: request an ACM certificate for
  `api.training.example.com` (regional, not `us-east-1`-locked like
  CloudFront's) and add an HTTPS listener on port 443, redirecting HTTP
  to HTTPS.
- **CI/CD**: trigger `register-task-definition` + `update-service`
  automatically from a GitHub Actions workflow on every push, instead of
  running those commands by hand.
- **Blue/green deploys**: replace the rolling `update-service` deployment
  with CodeDeploy's blue/green ECS deployment type for zero-downtime,
  instantly-reversible releases.
- **Alerting**: add CloudWatch alarms (Level 1, module 8) on ALB 5xx rate
  and ECS service CPU, wired to an SNS topic (module 4) for on-call
  notification.
- **WAF**: attach AWS WAF to the ALB or CloudFront distribution with a
  managed rule group, and observe blocked-request metrics.
- **Cost check**: use module 9's Cost Explorer grouped by tag to confirm
  this stack's actual daily cost matches your expectations before you
  leave it running for any length of time.
