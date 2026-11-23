---
title: "AWS Cheat Sheet: Quick Reference, Beginner to Advanced"
description: "A free AWS cheat sheet: 38 quick-reference tables from every level of the AWS Mastery Path, each linked to its full lesson."
---

# AWS Cheat Sheet

Quick-reference tables from **38 lessons** of the free AWS Mastery Path, in course order from beginner to advanced. Each section links to the full lesson with explanations and runnable examples.

## Level 1 · Entry

### [01 · Setup &amp; AWS CLI](level-1/01-setup-cli.md)

| Command / file | Purpose |
|---|---|
| `aws configure` | Interactively set access key, secret key, region, output format for the default profile. |
| `aws configure --profile NAME` | Same, but for a named profile. |
| `aws sts get-caller-identity` | Confirm who the CLI is currently authenticated as. |
| `~/.aws/credentials` | Stores access key / secret key per profile. |
| `~/.aws/config` | Stores region / output format per profile. |
| `--region` | Override the region for one command. |
| `--profile` | Override which credential set to use for one command. |
| `--output json\|table\|text` | Controls response formatting. |
| `--query "JMESPath"` | Filters/reshapes the JSON response client-side. |
| `AWS_PROFILE` env var | Sets the default profile for the whole shell session. |

### [02 · IAM &amp; Security Basics](level-1/02-iam-security-basics.md)

| Concept | CLI command |
|---|---|
| Create a group | `aws iam create-group --group-name NAME` |
| Attach managed policy to group | `aws iam attach-group-policy --group-name NAME --policy-arn ARN` |
| Add user to group | `aws iam add-user-to-group --group-name NAME --user-name USER` |
| Create a custom policy | `aws iam create-policy --policy-name NAME --policy-document file://FILE.json` |
| Create a role | `aws iam create-role --role-name NAME --assume-role-policy-document file://FILE.json` |
| Attach policy to role | `aws iam attach-role-policy --role-name NAME --policy-arn ARN` |
| Create instance profile | `aws iam create-instance-profile --instance-profile-name NAME` |
| Check what a user can do | `aws iam list-attached-user-policies --user-name USER` |
| Enable MFA | `aws iam enable-mfa-device --user-name USER --serial-number ARN --authentication-code1 X --authentication-code2 Y` |

### [03 · EC2 Compute](level-1/03-ec2-compute.md)

| Command | Purpose |
|---|---|
| `aws ec2 create-key-pair --key-name NAME` | Create an SSH key pair, prints the private key. |
| `aws ec2 create-security-group --group-name NAME --description DESC` | Create a firewall group. |
| `aws ec2 authorize-security-group-ingress --group-id ID --protocol tcp --port N --cidr CIDR` | Open a port. |
| `aws ec2 run-instances --image-id AMI --instance-type TYPE --key-name KEY --security-group-ids SG` | Launch an instance. |
| `aws ec2 describe-instances` | List/describe instances. |
| `aws ec2 wait instance-status-ok --instance-ids ID` | Block until an instance passes health checks. |
| `ssh -i KEY.pem ec2-user@IP` | Connect to an Amazon Linux instance. |
| `aws ec2 stop-instances` / `start-instances` | Pause/resume billing for compute (storage still billed). |
| `aws ec2 terminate-instances` | Delete the instance and its root volume. |

### [04 · S3 Storage](level-1/04-s3-storage.md)

| Command | Purpose |
|---|---|
| `aws s3 mb s3://NAME` | Create a bucket. |
| `aws s3 cp FILE s3://BUCKET/KEY` | Upload/download an object. |
| `aws s3 sync DIR s3://BUCKET/PREFIX` | Sync a local directory to/from S3. |
| `aws s3 ls s3://BUCKET/` | List objects. |
| `aws s3 rm s3://BUCKET/KEY` | Delete an object. |
| `aws s3api put-bucket-versioning --bucket B --versioning-configuration Status=Enabled` | Turn on versioning. |
| `aws s3api list-object-versions --bucket B` | List all versions of all objects. |
| `aws s3api put-bucket-policy --bucket B --policy file://p.json` | Attach a bucket policy. |
| `aws s3api put-public-access-block ...` | Opt in/out of S3's public-access safety blocks. |
| `aws s3api put-bucket-website --bucket B --website-configuration '{...}'` | Enable static website hosting. |

### [05 · VPC Networking Basics](level-1/05-vpc-networking-basics.md)

| Command | Purpose |
|---|---|
| `aws ec2 create-vpc --cidr-block CIDR` | Create a VPC. |
| `aws ec2 create-subnet --vpc-id ID --cidr-block CIDR --availability-zone AZ` | Create a subnet. |
| `aws ec2 create-internet-gateway` / `attach-internet-gateway` | Create and attach an IGW to a VPC. |
| `aws ec2 create-route-table --vpc-id ID` | Create a route table. |
| `aws ec2 create-route --route-table-id ID --destination-cidr-block 0.0.0.0/0 --gateway-id IGW` | Route all external traffic to the IGW. |
| `aws ec2 associate-route-table --route-table-id ID --subnet-id ID` | Bind a route table to a subnet. |
| `aws ec2 modify-subnet-attribute --subnet-id ID --map-public-ip-on-launch` | Auto-assign public IPs in a subnet. |
| `aws ec2 describe-vpcs` / `describe-subnets` / `describe-route-tables` | Inspect what you've built. |

### [06 · RDS Databases](level-1/06-rds-databases.md)

| Command | Purpose |
|---|---|
| `aws rds create-db-subnet-group` | Group subnets (2+ AZs) for RDS to place instances in. |
| `aws rds create-db-instance --engine postgres ...` | Provision a managed database instance. |
| `aws rds wait db-instance-available` | Block until the instance is ready. |
| `aws rds describe-db-instances` | Get endpoint address, port, status. |
| `aws rds restore-db-instance-to-point-in-time` | Restore to a new instance as of a past timestamp. |
| `aws rds modify-db-instance` | Change instance class, storage, etc. |
| `aws rds delete-db-instance --skip-final-snapshot` | Delete the instance (no final backup). |
| `--source-group` (on a security group rule) | Allow traffic only from members of another security group. |

### [07 · Lambda &amp; Serverless](level-1/07-lambda-serverless.md)

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

### [08 · CloudWatch Monitoring &amp; Logging](level-1/08-cloudwatch-monitoring.md)

| Command | Purpose |
|---|---|
| `aws cloudwatch list-metrics --namespace NS` | Discover what metrics exist. |
| `aws cloudwatch get-metric-statistics ...` | Pull numeric data points for a metric. |
| `aws logs describe-log-groups` | List log groups. |
| `aws logs tail /aws/lambda/NAME --follow` | Stream logs live. |
| `aws logs put-retention-policy --retention-in-days N` | Stop logs from accumulating forever. |
| `aws sns create-topic` / `subscribe` | Create a notification channel for alarms. |
| `aws cloudwatch put-metric-alarm ...` | Create an alarm that fires on a threshold. |
| `aws cloudwatch describe-alarms` | Check current alarm state. |
| `aws logs start-query` / `get-query-results` | Run a Logs Insights query. |

### [09 · CloudFormation &amp; IaC Intro](level-1/09-cloudformation-iac.md)

| Command | Purpose |
|---|---|
| `aws cloudformation deploy --template-file F --stack-name N --parameter-overrides K=V` | Create or update a stack from a template. |
| `aws cloudformation describe-stacks --stack-name N` | View a stack's status and outputs. |
| `aws cloudformation describe-stack-events --stack-name N` | See per-resource creation/update progress. |
| `aws cloudformation create-change-set` / `describe-change-set` | Preview changes before applying. |
| `aws cloudformation delete-stack --stack-name N` | Delete every resource the stack owns. |
| `aws cloudformation wait stack-delete-complete` | Block until deletion finishes. |
| `!Ref`, `!GetAtt`, `!Sub` | Core template intrinsic functions. |

### [10 · Capstone — Serverless Web App](level-1/10-capstone-project.md)

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

## Level 2 · Intermediate

### [01 · Containers on AWS (ECS/Fargate)](level-2/01-ecs-containers.md)

| Command | Purpose |
|---|---|
| `aws ecr create-repository --repository-name NAME` | Create a private image registry. |
| `aws ecr get-login-password \| docker login ...` | Authenticate Docker to ECR. |
| `aws ecs create-cluster --cluster-name NAME` | Create a cluster namespace. |
| `aws ecs register-task-definition --cli-input-json file://F` | Register a task definition revision. |
| `aws ecs create-service --launch-type FARGATE --network-configuration ...` | Run a task definition as a self-healing service. |
| `aws ecs update-service --task-definition FAMILY:REV` | Roll out a new task definition revision. |
| `aws ecs describe-services --cluster C --services S` | Check running vs. desired task count. |
| `aws logs tail /ecs/GROUP --since 10m` | Tail recent container logs. |

### [02 · Auto Scaling &amp; Load Balancing](level-2/02-autoscaling-load-balancing.md)

| Command | Purpose |
|---|---|
| `aws ec2 create-launch-template` | Define the AMI/instance type/security groups/user data for future instances. |
| `aws elbv2 create-load-balancer --type application --subnets ...` | Create an ALB across 2+ AZs. |
| `aws elbv2 create-target-group --health-check-path P` | Define a routable, health-checked target set. |
| `aws elbv2 create-listener --default-actions Type=forward,TargetGroupArn=...` | Route incoming traffic to a target group. |
| `aws autoscaling create-auto-scaling-group --target-group-arns ...` | Create a self-healing fleet wired to the ALB. |
| `aws autoscaling put-scaling-policy --policy-type TargetTrackingScaling` | Scale automatically on a metric target. |
| `aws elbv2 describe-target-health` | Check which instances are passing health checks. |
| `aws autoscaling start-instance-refresh` | Roll out a new launch template version gradually. |

### [03 · Route 53 &amp; DNS](level-2/03-route53-dns.md)

| Command | Purpose |
|---|---|
| `aws route53 create-hosted-zone --name DOMAIN --caller-reference X` | Create a DNS zone for a domain. |
| `aws route53 change-resource-record-sets --change-batch file://F` | Create/update/delete records (UPSERT/CREATE/DELETE). |
| `aws route53 get-change --id ID` | Check whether a record change has propagated to Route 53's servers. |
| `aws route53 list-resource-record-sets --hosted-zone-id Z` | List all records in a zone. |
| `aws route53 create-health-check` | Define an endpoint probe for failover routing. |
| `aws route53domains check-domain-availability` | Check if a domain name can be registered. |
| `dig NAME` / `nslookup NAME` | Query DNS directly to verify what resolvers actually see. |

### [04 · SNS &amp; SQS Messaging](level-2/04-sns-sqs-messaging.md)

| Command | Purpose |
|---|---|
| `aws sns create-topic --name NAME` | Create a pub/sub topic. |
| `aws sqs create-queue --attributes '{"RedrivePolicy": ...}'` | Create a queue with a dead-letter policy. |
| `aws sns subscribe --protocol sqs --notification-endpoint QUEUE_ARN` | Wire a queue to receive topic messages. |
| `aws sqs set-queue-attributes --attributes '{"Policy": ...}'` | Grant SNS permission to send to the queue. |
| `aws sns publish --message MSG --message-attributes ATTRS` | Publish a message to a topic. |
| `aws sqs receive-message --wait-time-seconds N` | Long-poll for messages. |
| `aws sqs delete-message --receipt-handle H` | Acknowledge successful processing. |
| `aws sns set-subscription-attributes --attribute-name FilterPolicy` | Restrict which messages a subscriber receives. |

### [05 · DynamoDB Deep Dive](level-2/05-dynamodb-deep-dive.md)

| Command | Purpose |
|---|---|
| `aws dynamodb create-table --key-schema HASH+RANGE --global-secondary-indexes [...]` | Create a table with a composite key and a GSI. |
| `aws dynamodb query --key-condition-expression ...` | Efficient, key-based read (base table or an index). |
| `aws dynamodb query --index-name NAME` | Query via a GSI/LSI instead of the base table. |
| `aws dynamodb batch-write-item --request-items file://F` | Write up to 25 items in one call (no conditions). |
| `aws dynamodb get-item --consistent-read` | Force a strongly consistent read (base table/LSI only). |
| `aws dynamodb update-table --billing-mode MODE` | Switch between on-demand and provisioned capacity. |
| `aws dynamodb update-table --stream-specification StreamEnabled=true,...` | Enable a change stream. |
| `aws dynamodb update-time-to-live` | Enable automatic item expiry. |

### [06 · Elastic Beanstalk](level-2/06-elastic-beanstalk.md)

| Command | Purpose |
|---|---|
| `aws elasticbeanstalk create-application --application-name N` | Create the logical app container. |
| `aws elasticbeanstalk create-application-version --source-bundle S3Bucket=B,S3Key=K` | Register a deployable build. |
| `aws elasticbeanstalk create-environment --solution-stack-name S --version-label V` | Provision a running environment from a version. |
| `aws elasticbeanstalk describe-environments` | Check `Status`/`Health`/`CNAME`. |
| `aws elasticbeanstalk update-environment --version-label V` | Roll out a new version to an existing environment. |
| `aws elasticbeanstalk swap-environment-cnames` | Atomically swap two environments' public URLs (blue/green). |
| `aws elasticbeanstalk terminate-environment --environment-name N` | Tear down an environment's underlying resources. |

### [07 · Secrets Manager &amp; Parameter Store](level-2/07-secrets-parameter-store.md)

| Command | Purpose |
|---|---|
| `aws secretsmanager create-secret --secret-string JSON` | Store a new secret value. |
| `aws secretsmanager get-secret-value --secret-id ID` | Retrieve the current secret value. |
| `aws secretsmanager put-secret-value` | Write a new version of a secret. |
| `aws secretsmanager rotate-secret --rotation-lambda-arn ARN --rotation-rules ...` | Enable scheduled automatic rotation. |
| `aws ssm put-parameter --type SecureString` | Store an encrypted config value. |
| `aws ssm get-parameter --with-decryption` | Read a `SecureString` parameter's plaintext. |
| `aws ssm get-parameters-by-path --recursive` | Fetch an entire config namespace at once. |

### [08 · CloudFront &amp; CDN Basics](level-2/08-cloudfront-cdn.md)

| Command | Purpose |
|---|---|
| `aws cloudfront create-origin-access-control` | Create OAC so CloudFront can read a private S3 bucket. |
| `aws cloudfront create-distribution --distribution-config file://F` | Create a CDN distribution. |
| `aws s3api put-bucket-policy` | Restrict the S3 origin to only the CloudFront distribution. |
| `aws cloudfront get-distribution --id ID` | Check deployment status and config. |
| `aws cloudfront create-invalidation --paths "/path"` | Force edges to re-fetch specific paths. |
| `aws acm request-certificate --region us-east-1` | Request an HTTPS certificate (must be `us-east-1` for CloudFront). |
| `aws cloudfront update-distribution` | Change config (origins, aliases, cache behaviors). |

### [09 · Cost Management &amp; Billing](level-2/09-cost-management-billing.md)

| Command | Purpose |
|---|---|
| `aws budgets create-budget --notifications-with-subscribers file://F` | Create a cost budget with an email alert threshold. |
| `aws ce get-cost-and-usage --group-by Type=DIMENSION,Key=SERVICE` | See historical spend broken down by service. |
| `aws ce get-cost-and-usage --group-by Type=TAG,Key=NAME` | See spend broken down by an activated cost allocation tag. |
| `aws ce get-cost-forecast` | Project future spend from historical trend. |
| `aws resourcegroupstaggingapi tag-resources` | Apply tags across resources for later cost attribution. |
| `aws cur put-report-definition` | Set up a granular daily/hourly Cost and Usage Report to S3. |

### [10 · Project — Scalable Web App](level-2/10-project-scalable-web-app.md)

| Command | Purpose |
|---|---|
| `aws dynamodb create-table --global-secondary-indexes ...` | Data layer with a "recent items" query pattern. |
| `aws ecs create-service --load-balancers [...]` | Wire a Fargate service directly to an ALB target group. |
| `aws application-autoscaling register-scalable-target --service-namespace ecs` | Enable ECS service auto scaling. |
| `aws cloudfront create-distribution` | CDN in front of the static frontend. |
| `aws route53 change-resource-record-sets` | Point real domain names at the ALB and CloudFront. |
| `aws ecs stop-task` | Manually kill a task to test self-healing. |

## Level 3 · Advanced

### [Advanced VPC (Peering, Transit Gateway, VPN)](level-3/01-advanced-vpc.md)

| Tool | Topology | Transitive? | Typical use |
|---|---|---|---|
| VPC Peering | Point-to-point | No | 2-4 VPCs, simple mesh |
| Transit Gateway | Hub-and-spoke | Yes (via TGW route tables) | 5+ VPCs, multi-account |
| Site-to-Site VPN | On-prem ↔ AWS | N/A | Quick on-prem connectivity |
| Direct Connect (L4) | On-prem ↔ AWS | N/A | Dedicated, high-bandwidth |

### [Kubernetes on AWS (EKS)](level-3/02-eks-kubernetes.md)

| Command | Purpose |
|---|---|
| `aws eks create-cluster` | Provision the control plane |
| `aws eks update-kubeconfig` | Configure local `kubectl` |
| `aws eks create-nodegroup` | Add EC2-backed workers |
| `aws eks create-fargate-profile` | Add serverless pod scheduling |
| `kubectl get nodes` | List workers the control plane sees |
| `kubectl apply -f <file>` | Create/update resources from a manifest |
| `kubectl logs <pod>` | Tail container logs |

### [Step Functions &amp; Orchestration](level-3/03-step-functions-orchestration.md)

| Command | Purpose |
|---|---|
| `aws stepfunctions create-state-machine` | Deploy a new definition |
| `aws stepfunctions update-state-machine` | Update an existing one |
| `aws stepfunctions start-execution` | Trigger a run |
| `aws stepfunctions describe-execution` | Check status/output |
| `aws stepfunctions get-execution-history` | Full step-by-step trace |
| `aws stepfunctions stop-execution` | Cancel a running execution |

### [Advanced IAM (SCPs, Permission Boundaries)](level-3/04-advanced-iam.md)

| Mechanism | Applies to | Can grant permissions? |
|---|---|---|
| Identity-based policy | User/role | Yes |
| Resource-based policy | Resource (e.g. S3 bucket) | Yes |
| Permission boundary | User/role | No — caps only |
| SCP | Account/OU | No — caps only |
| `sts:AssumeRole` + `ExternalId` | Cross-account | N/A (auth mechanism) |

### [Terraform on AWS](level-3/05-terraform-on-aws.md)

| Command | Purpose |
|---|---|
| `terraform init` | Download providers, configure backend |
| `terraform plan` | Preview changes |
| `terraform apply` | Apply changes |
| `terraform destroy` | Tear down everything in state |
| `terraform state list` | List resources Terraform is tracking |
| `terraform import <addr> <id>` | Bring an existing resource under management |
| `terraform fmt` / `validate` | Format / syntax-check HCL |

### [CI/CD (CodePipeline, CodeBuild, CodeDeploy)](level-3/06-cicd-codepipeline.md)

| Command | Purpose |
|---|---|
| `aws codepipeline create-pipeline` | Define a new pipeline |
| `aws codepipeline start-pipeline-execution` | Trigger a run manually |
| `aws codebuild start-build` | Run a build outside a pipeline |
| `aws codebuild batch-get-builds` | Check build status/logs |
| `aws deploy create-deployment` | Trigger a CodeDeploy deployment directly |
| `aws codepipeline get-pipeline-state` | See which stage is currently running |

### [Multi-Region &amp; Disaster Recovery](level-3/07-multi-region-disaster-recovery.md)

| Task | Command |
|---|---|
| Cross-region snapshot copy | `aws rds copy-db-snapshot --source-region ...` |
| Cross-region S3 replication | `aws s3api put-bucket-replication` |
| Create read replica in another region | `aws rds create-db-instance-read-replica` |
| Promote replica during failover | `aws rds promote-read-replica` |
| DNS failover | `aws route53 change-resource-record-sets` with `Failover` policy |
| Health check | `aws route53 create-health-check` |

### [Advanced Monitoring (X-Ray &amp; Tracing)](level-3/08-advanced-monitoring-xray.md)

| Command | Purpose |
|---|---|
| `aws lambda update-function-configuration --tracing-config` | Enable tracing on Lambda |
| `aws apigateway update-stage ... tracingEnabled` | Enable tracing on API Gateway |
| `aws xray get-trace-summaries` | Query recent traces |
| `aws xray batch-get-traces` | Fetch full trace detail |
| `aws xray get-service-graph` | Fetch the service map data |
| `aws xray put-trace-segments` | Manually submit segments (custom instrumentation) |

### [Security Deep Dive (GuardDuty, WAF, Security Hub)](level-3/09-security-deep-dive.md)

| Command | Purpose |
|---|---|
| `aws guardduty create-detector --enable` | Turn on threat detection |
| `aws guardduty list-findings` | List current findings |
| `aws wafv2 create-web-acl` | Create a Web ACL |
| `aws wafv2 associate-web-acl` | Attach it to ALB/CloudFront/API Gateway |
| `aws securityhub enable-security-hub` | Turn on centralized findings |
| `aws securityhub get-findings` | Query aggregated findings |

## Level 4 · Master

### [AWS Well-Architected Framework](level-4/01-well-architected-framework.md)

| Command | Purpose |
|---|---|
| `aws wellarchitected create-workload` | Register a workload for review |
| `aws wellarchitected update-answer` | Record answers to pillar questions |
| `aws wellarchitected list-lens-review-improvements` | List HRIs/MRIs with remediation guidance |
| `aws wellarchitected create-milestone` | Snapshot current risk state |
| `aws wellarchitected get-lens-review-report` | Generate a PDF summary |

### [Advanced Serverless (EventBridge at Scale)](level-4/02-advanced-serverless-eventbridge.md)

| Command | Purpose |
|---|---|
| `aws events create-event-bus` | Create a custom bus |
| `aws events put-events` | Publish an event |
| `aws events put-rule` / `put-targets` | Route events by pattern |
| `aws schemas create-registry` | Track event schemas |
| `aws events create-archive` | Enable replay capability |
| `aws events start-replay` | Re-deliver past events |

### [Data Lakes &amp; Analytics (S3, Athena, Glue, Redshift)](level-4/03-data-lakes-analytics.md)

| Task | Command |
|---|---|
| Catalog a database | `aws glue create-database` |
| Discover schema/partitions | `aws glue create-crawler` + `start-crawler` |
| Ad-hoc SQL over S3 | `aws athena start-query-execution` |
| Transform data at scale | `aws glue create-job` + `start-job-run` |
| Provisioned warehouse | `aws redshift-serverless create-workgroup` |
| Query lake from Redshift | `CREATE EXTERNAL SCHEMA ... FROM DATA CATALOG` |

### [Kubernetes at Scale (EKS Production Patterns)](level-4/04-eks-at-scale.md)

| Task | Tool/command |
|---|---|
| Auto-provision right-sized nodes | Karpenter `NodePool` |
| Scale existing ASG-backed node groups | Cluster Autoscaler |
| Continuous Git-driven deployment | Argo CD `Application` |
| Cap team resource usage | `ResourceQuota` |
| Isolate namespace traffic | `NetworkPolicy` |
| Spot capacity in managed node groups | `--capacity-type SPOT` |

### [Multi-Account Strategy (Organizations, Control Tower)](level-4/05-multi-account-strategy.md)

| Command | Purpose |
|---|---|
| `aws organizations create-organization` | Bootstrap an Organization |
| `aws organizations create-account` | Provision a new member account |
| `aws organizations move-account` | Reorganize OU structure |
| `aws controltower create-account-factory-account` | Provision a governed account |
| `aws ram create-resource-share` | Share a resource across accounts |
| `aws cloudtrail create-trail --is-organization-trail` | Org-wide audit logging |

### [Cost Optimization at Scale](level-4/06-cost-optimization-at-scale.md)

| Task | Command |
|---|---|
| Get Savings Plan recommendation | `aws ce get-savings-plans-purchase-recommendation` |
| Purchase a Savings Plan | `aws savingsplans create-savings-plan` |
| Request diversified spot capacity | `aws ec2 request-spot-fleet` |
| Get rightsizing recommendations | `aws compute-optimizer get-ec2-instance-recommendations` |
| Group costs by tag | `aws ce get-cost-and-usage --group-by Type=TAG` |
| Detect spend anomalies | `aws ce create-anomaly-monitor` |
| Tier S3 storage automatically | `aws s3api put-bucket-lifecycle-configuration` |

### [Advanced Networking (Direct Connect, Global Accelerator)](level-4/07-advanced-networking.md)

| Command | Purpose |
|---|---|
| `aws directconnect create-connection` | Request a physical DX port |
| `aws directconnect create-private-virtual-interface` | Route DX traffic to a VPC |
| `aws directconnect create-public-virtual-interface` | Route DX traffic to AWS public endpoints |
| `aws globalaccelerator create-accelerator` | Create a global static-IP entry point |
| `aws globalaccelerator create-endpoint-group` | Add a regional target with health checks |

### [Compliance &amp; Governance (Config, Audit Manager)](level-4/08-compliance-governance.md)

| Command | Purpose |
|---|---|
| `aws configservice put-configuration-recorder` | Start tracking resource configs |
| `aws configservice put-config-rule` | Add a compliance rule |
| `aws configservice get-compliance-details-by-config-rule` | Check compliance status |
| `aws configservice put-remediation-configurations` | Auto-fix non-compliant resources |
| `aws configservice put-conformance-pack` | Deploy a framework's rule bundle |
| `aws auditmanager create-assessment` | Start continuous evidence collection |

### [Chaos Engineering &amp; Resilience Testing](level-4/09-chaos-engineering-resilience.md)

| Command | Purpose |
|---|---|
| `aws fis create-experiment-template` | Define an experiment |
| `aws fis start-experiment` | Run it |
| `aws fis get-experiment` | Check status/results |
| `aws fis stop-experiment` | Manually abort a running experiment |
| `aws fis list-experiments` | Review experiment history |

---

Go deeper with the full [AWS Mastery Path](index.md) curriculum or test yourself with the [interview questions](interview-questions.md).
