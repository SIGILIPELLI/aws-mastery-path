---
title: "AWS Interview Questions and Answers (180 Questions)"
description: "180 AWS interview questions with answers and explanations, from beginner to advanced, organised by level. Free, from the AWS Mastery Path."
---

# AWS Interview Questions and Answers

**180 questions** with the correct answer and a short explanation, organised from beginner (Level 1) to advanced (Level 4). Every question comes from a lesson in the free AWS Mastery Path, so you can open the lesson to go deeper.

!!! tip "Practise, don't just read"
    Each lesson also ends with an interactive quiz, and the [final quiz](quiz.md) tests all four levels. Try to answer each question below before reading the answer.

## Level 1 · Entry

### [01 · Setup &amp; AWS CLI](level-1/01-setup-cli.md)

**Q1.** What does the course say to do with the root user after creating the account?

**Answer:** Use it only to create your first IAM user, then stop using it for daily work

The root user cannot be restricted by any policy, so a leak gives full control of the account and bill.

**Q2.** When is the secret access key shown?

**Answer:** Only once, at creation, so save it in a password manager

Create the key with the CLI use case and store it safely.

**Q3.** Why attach AdministratorAccess to your own IAM user in this lesson?

**Answer:** It is your day-to-day login, not a service credential; least privilege for service roles comes later

The policy is broad on purpose for this personal user.

**Q4.** Which two beginner mistakes does the module aim to prevent?

**Answer:** Using the root user day-to-day, and not understanding regions/profiles

Confusion about regions and profiles causes 'where did my resource go?' bugs.

### [02 · IAM &amp; Security Basics](level-1/02-iam-security-basics.md)

**Q5.** Which IAM identity uses temporary credentials and is assumed by users or AWS services?

**Answer:** Role

There is no long-lived secret to leak.

**Q6.** Why attach policies to a group rather than to each user?

**Answer:** You manage permissions for a team without repeating yourself

Add users to the group and they inherit its policies.

**Q7.** What are the parts of a policy statement?

**Answer:** Effect (Allow/Deny), Actions and Resources

The example grants read-only access to exactly one bucket.

**Q8.** Is IAM region-scoped?

**Answer:** No, it is global

It's the most important service to understand before provisioning anything else.

### [03 · EC2 Compute](level-1/03-ec2-compute.md)

**Q9.** What is an AMI?

**Answer:** A template of OS and pre-installed software an instance boots from

The instance type is the hardware profile, such as t3.micro.

**Q10.** Why chmod 400 the .pem private key?

**Answer:** SSH refuses a private key file that other users could read

AWS cannot give the private key back, so losing it means creating a new key pair.

**Q11.** What does it mean that security groups are stateful?

**Answer:** A response to an allowed inbound request is automatically allowed back out

You rarely need explicit outbound rules for typical request/response traffic.

**Q12.** Why allow SSH only from your own IP /32 rather than 0.0.0.0/0?

**Answer:** It's safer: only your address can attempt to connect

HTTP on port 80 from anywhere is the web-server exception in the lesson.

### [04 · S3 Storage](level-1/04-s3-storage.md)

**Q13.** What must be true of an S3 bucket name?

**Answer:** Globally unique across all of AWS

Not just unique in your own account.

**Q14.** What does aws s3 sync do?

**Answer:** Uploads only changed or new files from a directory

It's the efficient way to push a whole local directory.

**Q15.** What does enabling versioning do?

**Answer:** Keeps every version of an object instead of overwriting, at the cost of storing (and billing) each version

It protects against accidental deletes and overwrites.

**Q16.** When do you use aws s3api instead of aws s3?

**Answer:** For lower-level operations like bucket policies and versioning configuration

aws s3 is the friendly high-level command set.

### [05 · VPC Networking Basics](level-1/05-vpc-networking-basics.md)

**Q17.** What makes a subnet public?

**Answer:** It has a route to an Internet Gateway

Resources there can have public IPs; private subnets have no direct route to the internet.

**Q18.** What does a NAT Gateway allow?

**Answer:** Private-subnet resources to make outbound internet calls without being reachable from outside

It's used for databases and internal services that need to reach out.

**Q19.** Why did Module 3's EC2 instance work without networking setup?

**Answer:** It landed in the default VPC, pre-wired with public subnets

New accounts get a default VPC in every region.

**Q20.** A /16 VPC CIDR provides how many addresses?

**Answer:** 65,536

A /24 subnet provides 256.

### [06 · RDS Databases](level-1/06-rds-databases.md)

**Q21.** What does RDS handle for you?

**Answer:** Patching, backups and failover

You focus on schema and queries instead of database administration.

**Q22.** Why does RDS require a DB subnet group with two subnets in different AZs?

**Answer:** It requires at least two AZs even for a single-AZ instance

The lesson creates a second private subnet in us-east-1b.

**Q23.** What does --source-group do in the database security group rule?

**Answer:** Allows traffic from anything in the app security group instead of specific IPs

It's the standard 'app tier talks to data tier' pattern.

**Q24.** What is the trade-off of RDS versus installing a database on EC2 yourself?

**Answer:** Less low-level control: you can't SSH into the underlying OS

In exchange you get automatic backups with point-in-time recovery, patching and Multi-AZ failover.

### [07 · Lambda &amp; Serverless](level-1/07-lambda-serverless.md)

**Q25.** What do you pay for in Lambda?

**Answer:** The milliseconds your code actually executes

It scales from zero to thousands of concurrent invocations.

**Q26.** What determines what a Lambda function can access?

**Answer:** Its execution role, not the caller's identity

The role is assumed by the Lambda service via a trust policy.

**Q27.** What is a cold start?

**Answer:** The latency of provisioning a fresh execution environment for the first invocation or after scaling up

It's one of the core Lambda concepts.

**Q28.** What is the bare minimum policy every Lambda function needs?

**Answer:** AWSLambdaBasicExecutionRole, to write logs

Add more permissions only as the function actually needs them.

### [08 · CloudWatch Monitoring &amp; Logging](level-1/08-cloudwatch-monitoring.md)

**Q29.** What are CloudWatch namespaces?

**Answer:** Groups of metrics such as AWS/Lambda, AWS/EC2 and AWS/RDS

Every AWS service publishes metrics grouped into namespaces.

**Q30.** Where does Lambda's print()/logging output land?

**Answer:** A CloudWatch log group /aws/lambda/&lt;function-name&gt;, created automatically

No setup is required on your part.

**Q31.** Which command tails a Lambda's most recent log events live?

**Answer:** aws logs tail /aws/lambda/training-hello --follow

It's great for live-debugging.

**Q32.** What does a CloudWatch alarm do?

**Answer:** Notifies you when a metric crosses a threshold

The module sets one up alongside reading metrics and logs.

### [09 · CloudFormation &amp; IaC Intro](level-1/09-cloudformation-iac.md)

**Q33.** What is the difference between a CloudFormation template and a stack?

**Answer:** A template is the YAML/JSON description; a stack is the running set of resources created from it

CloudFormation tracks the stack's resources together so it can update or delete them as a unit.

**Q34.** What does 'drift' mean in the IaC table?

**Answer:** Console clicks that change resources invisibly; diffing against the template reveals them

Manual changes are invisible without an IaC record.

**Q35.** How do you tear down everything a template created?

**Answer:** One command deletes the stack and its resources

Manual teardown means remembering every resource.

**Q36.** What does !Ref BucketName do in the template?

**Answer:** Inserts the value of the BucketName parameter

Parameters make templates reusable across environments.

### [10 · Capstone — Serverless Web App](level-1/10-capstone-project.md)

**Q37.** What are the pieces of the capstone's architecture?

**Answer:** S3 static site, API Gateway, Lambda and DynamoDB

It's a notes app where visitors add and read notes.

**Q38.** Why PAY_PER_REQUEST billing for the DynamoDB table?

**Answer:** It charges only for reads/writes actually made, with no capacity to provision, right for low traffic

On-demand billing is the right default for a training app.

**Q39.** Why does the lesson stress the Teardown section?

**Answer:** Free tier is free below a usage threshold, and only 12 months on some resources, so leftover resources can bill you

Tear down every time you finish working, not just at the end of the course.

**Q40.** How does DynamoDB compare with RDS conceptually?

**Answer:** Both are managed with no server to patch, but DynamoDB is key-value/document shaped rather than relational

It's new in this module but parallel to Module 6.

## Level 2 · Intermediate

### [01 · Containers on AWS (ECS/Fargate)](level-2/01-ecs-containers.md)

**Q41.** What does the Fargate launch type remove?

**Answer:** Provisioning and patching the underlying EC2 instances

You specify vCPU and memory per task and AWS handles the host.

**Q42.** What is a task definition?

**Answer:** A JSON blueprint of images, CPU/memory, ports, roles and logging

A task is a running instance of it.

**Q43.** What does an ECS service do?

**Answer:** Keeps a desired number of tasks running, replacing any that die

It can also register tasks with a load balancer.

**Q44.** Why do CI pipelines re-authenticate to ECR each run?

**Answer:** The get-login-password token is valid for 12 hours

They don't cache the token.

### [02 · Auto Scaling &amp; Load Balancing](level-2/02-autoscaling-load-balancing.md)

**Q45.** What does an Auto Scaling Group do?

**Answer:** Keeps a fleet of identical instances running, replaces failed ones and scales with demand

It maintains the fleet between min/max size.

**Q46.** What does an ALB health check do?

**Answer:** Probes targets, and failing ones stop receiving traffic

It's defined on the target group.

**Q47.** Why must UserData be base64-encoded in the JSON launch-template example?

**Answer:** The CLI does not encode it for you when embedded in a JSON string

Each instance runs the script once at boot.

**Q48.** What is a launch template?

**Answer:** A reusable blueprint of AMI, instance type, security groups and user data

The ASG creates instances from it.

### [03 · Route 53 &amp; DNS](level-2/03-route53-dns.md)

**Q49.** Why use an alias record rather than a CNAME for a zone apex?

**Answer:** Alias records work at the zone apex and point at AWS resources like an ALB, CloudFront or S3 website

It's a Route 53-specific record type.

**Q50.** What is a hosted zone?

**Answer:** A container for DNS records for one domain, with its own 4 name servers

For a real subdomain you'd add those name servers as NS records at your registrar.

**Q51.** What does TTL control?

**Answer:** How long resolvers cache a record before re-querying

Time-to-live.

**Q52.** What is --caller-reference for?

**Answer:** A unique value per call so retried requests aren't treated as duplicates

A timestamp works.

### [04 · SNS &amp; SQS Messaging](level-2/04-sns-sqs-messaging.md)

**Q53.** How do SNS and SQS differ?

**Answer:** SNS is pub/sub fan-out to many subscribers; SQS is a queue where each message is processed by one consumer

Combining SNS → multiple SQS queues is the fan-out pattern.

**Q54.** What does the visibility timeout do?

**Answer:** Hides a received-but-not-deleted message from other consumers to avoid double-processing

The lesson sets it to 30 seconds.

**Q55.** What does maxReceiveCount: 5 mean?

**Answer:** A message received and left unprocessed 5 times moves to the dead-letter queue

The DLQ stops failing messages blocking the queue forever.

**Q56.** How does a FIFO queue differ from a standard queue?

**Answer:** Ordered, exactly-once processing (.fifo suffix) versus at-least-once, best-effort order

Standard queues are the default.

### [05 · DynamoDB Deep Dive](level-2/05-dynamodb-deep-dive.md)

**Q57.** What does a composite key of customerId (partition) + orderId (sort) enable?

**Answer:** Fetching all orders for one customer in a single efficient query

Items sharing a partition key are stored together and range-queryable by sort key.

**Q58.** What is a GSI?

**Answer:** An alternate partition/sort key over the same table, queryable independently with its own eventually-consistent copy

It's the main tool for querying by something other than the primary key.

**Q59.** How does an LSI differ from a GSI?

**Answer:** Uses the same partition key with an alternate sort key, created only at table creation, supporting strongly consistent reads

From the lesson's concept table.

**Q60.** When is PROVISIONED capacity cheaper than on-demand?

**Answer:** At steady high volume

PAY_PER_REQUEST charges per read/write; PROVISIONED reserves RCU/WCU.

### [06 · Elastic Beanstalk](level-2/06-elastic-beanstalk.md)

**Q61.** What does Elastic Beanstalk do?

**Answer:** Provisions and manages EC2, ASG, ALB and security groups from a single application upload

You deploy code; Beanstalk manages the servers, while still allowing access to the underlying resources.

**Q62.** What is an application version?

**Answer:** One uploaded, deployable build (a zip in S3) with a kept history

An environment runs one version.

**Q63.** What are .ebextensions?

**Answer:** Config files in your bundle that customize the environment's resources

They can add packages, config and resources.

**Q64.** What do the web server and worker environment tiers do?

**Answer:** Web handles HTTP traffic; worker processes an SQS queue in the background

Choose per environment.

### [07 · Secrets Manager &amp; Parameter Store](level-2/07-secrets-parameter-store.md)

**Q65.** Why avoid putting a database password in a Lambda environment variable?

**Answer:** It's visible to anyone with read access to the function configuration and never rotates

Use Secrets Manager or Parameter Store instead.

**Q66.** What does Secrets Manager offer that plain Parameter Store focuses less on?

**Answer:** Secrets with built-in automatic rotation

Rotation is done via a Lambda function on a schedule.

**Q67.** What does get-secret-value return by default?

**Answer:** The AWSCURRENT version

Every write creates a new version, and prior versions remain retrievable.

**Q68.** What does a SecureString parameter use?

**Answer:** KMS encryption

String and StringList are plain.

### [08 · CloudFront &amp; CDN Basics](level-2/08-cloudfront-cdn.md)

**Q69.** What does CloudFront do?

**Answer:** Caches content at edge locations so repeat requests are served near the visitor

It also provides managed HTTPS in front of an S3 bucket.

**Q70.** What does Origin Access Control (OAC) allow?

**Answer:** CloudFront to authenticate to a private S3 bucket so the bucket stays non-public

This is the CloudFront-recommended pattern, unlike the Level 1 public bucket policy.

**Q71.** What is an invalidation?

**Answer:** A request to purge specific paths from edge caches before the TTL expires

It's used when content changes before the cache would naturally refresh.

**Q72.** What is a cache behavior?

**Answer:** Per-path-pattern rules: origin, what varies the cache key, and TTLs

It lives inside the distribution configuration.

### [09 · Cost Management &amp; Billing](level-2/09-cost-management-billing.md)

**Q73.** What does an AWS Budget do when the threshold is crossed?

**Answer:** Only notifies, unless you wire it to an automated action

Crossing 100% does not stop anything from running or accruing cost.

**Q74.** What is Cost Explorer?

**Answer:** A queryable view of historical spend, groupable by service, tag or account

It also provides forecasts.

**Q75.** What does a cost allocation tag enable?

**Answer:** Attributing spend to a project, team or environment in billing reports

For example Project=training, activated for billing.

**Q76.** What is the Cost and Usage Report?

**Answer:** The most granular billing export, one row per line item, dumped to S3

It's a CSV or Parquet dump.

### [10 · Project — Scalable Web App](level-2/10-project-scalable-web-app.md)

**Q77.** How does the Level 2 project differ from the Level 1 capstone?

**Answer:** Containerized, load-balanced and globally distributed rather than serverless

It uses ECS/Fargate, ALB, DynamoDB, Route 53 and CloudFront.

**Q78.** How does the API reach DynamoDB without credentials?

**Answer:** Via an IAM task role

Secrets Manager would only hold third-party API keys.

**Q79.** What serves the static frontend?

**Answer:** CloudFront in front of a private S3 bucket via OAC

It's cached at the edge.

**Q80.** What does the byRecency GSI support?

**Answer:** Querying recent notes across all users

The base key is userId + noteId.

## Level 3 · Advanced

### [Advanced VPC (Peering, Transit Gateway, VPN)](level-3/01-advanced-vpc.md)

**Q81.** What does it mean that VPC peering is non-transitive?

**Answer:** If A peers with B and B with C, A cannot reach C through B

Each pair needs its own connection.

**Q82.** After accepting a peering connection, what else is required?

**Answer:** Route table entries on each side and security groups allowing the traffic

Accepting isn't enough.

**Q83.** Why use Transit Gateway instead of full-mesh peering?

**Answer:** Peering grows O(n²) (10 VPCs need 45 connections); TGW is hub-and-spoke with one attachment per VPC

TGW also supports multicast and cross-region peering.

**Q84.** What must be true of VPC CIDR ranges to connect them?

**Answer:** They must not overlap

Peering and Transit Gateway cannot route between colliding ranges.

### [Kubernetes on AWS (EKS)](level-3/02-eks-kubernetes.md)

**Q85.** When should you choose EKS over ECS?

**Answer:** For multi-cloud portability, existing Kubernetes expertise or ecosystem tools that assume Kubernetes APIs

ECS has a lower learning curve but is AWS-only.

**Q86.** How is the EKS control plane billed?

**Answer:** A flat hourly rate per cluster regardless of size, plus your worker compute

It also takes 10–15 minutes to provision.

**Q87.** What does kubectl authentication rely on after update-kubeconfig?

**Answer:** aws eks get-token, so it fails if your AWS CLI credentials expire

No long-lived credentials are stored in ~/.kube/config.

**Q88.** What does a managed node group handle?

**Answer:** AMI selection, patching and draining

For serverless pods, use Fargate profiles instead.

### [Step Functions &amp; Orchestration](level-3/03-step-functions-orchestration.md)

**Q89.** Why use Step Functions instead of chaining Lambdas in code?

**Answer:** The flow, retries and error handling become a visible first-class resource, not buried in code

It also doesn't bill compute while waiting.

**Q90.** What language defines a state machine?

**Answer:** Amazon States Language (ASL), a JSON dialect

The example defines ValidateOrder → IsValid → ProcessPayment → ShipOrder.

**Q91.** What do Retry and Catch provide?

**Answer:** Per-state exponential backoff and error branching without try/except code

For example IntervalSeconds 2, MaxAttempts 3, BackoffRate 2.0.

**Q92.** What does the Choice state do?

**Answer:** Branches on a condition, e.g. $.valid being true, with a Default branch

Invalid orders go to RejectOrder.

### [Advanced IAM (SCPs, Permission Boundaries)](level-3/04-advanced-iam.md)

**Q93.** What can a Service Control Policy do?

**Answer:** Set a ceiling on what an account or OU can do; it never grants permissions

A user with AdministratorAccess still can't do what an SCP denies.

**Q94.** What does a permission boundary cap?

**Answer:** The maximum permissions of an individual role or user, even if admin policies are attached

It lets teams create roles without privilege escalation.

**Q95.** Effective permissions are the ____ of the attached policy and the boundary.

**Answer:** intersection

AdministratorAccess on a bounded role is still capped at S3, Lambda and Logs.

**Q96.** What does NotAction with Deny in the region SCP do?

**Answer:** Denies everything except a short list of global services, outside the approved regions

Common pattern for region lockdown.

### [Terraform on AWS](level-3/05-terraform-on-aws.md)

**Q97.** Which Terraform command previews changes before applying them?

**Answer:** terraform plan

It is the equivalent of CloudFormation change sets; always read it before apply.

**Q98.** What is the biggest conceptual shift from CloudFormation?

**Answer:** Terraform uses an explicit .tfstate file that you own

Losing or corrupting it leads to Terraform and AWS disagreeing about what exists.

**Q99.** Why store state remotely (S3 with DynamoDB locking) in production?

**Answer:** Local state breaks when two people apply at once

Locking prevents concurrent applies.

**Q100.** How does Terraform differ from CloudFormation in rollback on failure?

**Answer:** Terraform has no built-in rollback (manual), CloudFormation rolls back automatically

From the comparison table.

### [CI/CD (CodePipeline, CodeBuild, CodeDeploy)](level-3/06-cicd-codepipeline.md)

**Q101.** What are the roles of CodePipeline, CodeBuild and CodeDeploy?

**Answer:** Orchestration, build/test and deployment respectively

Together they form AWS's native pipeline.

**Q102.** Where does CodeBuild read its commands from?

**Answer:** buildspec.yml in the repo root

It has install, pre_build, build and post_build phases.

**Q103.** Why tag images with the commit SHA (CODEBUILD_RESOLVED_SOURCE_VERSION) instead of latest?

**Answer:** It makes rollbacks to a specific prior build possible

CodeBuild injects that variable automatically.

**Q104.** What stages does the example pipeline move an artifact through?

**Answer:** Source → Build → Deploy

CodePipeline defines the stages.

### [Multi-Region &amp; Disaster Recovery](level-3/07-multi-region-disaster-recovery.md)

**Q105.** What is RTO?

**Answer:** How long you can tolerate being down

RPO is how much data you can tolerate losing.

**Q106.** Which DR strategy keeps only the database running in the DR region with compute scaled to zero?

**Answer:** Pilot Light

On failover you promote the replica and scale up compute.

**Q107.** Which is the cheapest strategy and what is its RTO?

**Answer:** Backup &amp; Restore, with hours of RTO

Nothing runs in the DR region until a declared disaster.

**Q108.** How should you pick a DR strategy?

**Answer:** Work backward from business requirements, not default to the most expensive

Costs rise with faster recovery.

### [Advanced Monitoring (X-Ray &amp; Tracing)](level-3/08-advanced-monitoring-xray.md)

**Q109.** What does X-Ray provide that per-resource logs don't?

**Answer:** Distributed tracing: one trace ID follows a request across every service it touches

Segments sharing a trace ID are stitched into one trace shown as a service map.

**Q110.** What does Mode=PassThrough do on Lambda?

**Answer:** Traces only if the incoming request already carries a trace header

Mode=Active traces invocations subject to the sampling rule.

**Q111.** What does patch_all() do?

**Answer:** Auto-instruments libraries such as boto3 and requests so calls appear as subsegments

A DynamoDB get_item shows up with its own latency.

**Q112.** What does filter-expression 'service("validateOrder") { fault }' find?

**Answer:** Traces where that service raised an error

You can also filter by response time, status or annotations.

### [Security Deep Dive (GuardDuty, WAF, Security Hub)](level-3/09-security-deep-dive.md)

**Q113.** What does GuardDuty analyze?

**Answer:** VPC Flow Logs, DNS logs and CloudTrail events against threat intelligence and anomaly models, with no agents to install

Findings are scored 1–10 by severity.

**Q114.** How does WAF differ from a security group?

**Answer:** WAF inspects HTTP request content (SQLi patterns, rate limits, geo blocks); security groups filter by IP/port

WAF attaches to CloudFront, ALB or API Gateway.

**Q115.** What does Security Hub provide?

**Answer:** Centralized findings

It aggregates findings from services like GuardDuty.

**Q116.** How can high-severity GuardDuty findings trigger automated response?

**Answer:** Route them to EventBridge with a rule on severity

The example matches severity &gt;= 7.

### [Project — Multi-Tier CI/CD Pipeline](level-3/10-project-cicd-pipeline.md)

**Q117.** Why does the project promote the same image from staging to prod by SHA tag?

**Answer:** The exact image tested in staging is what is promoted, never a rebuild

Two clusters share one ECR repository.

**Q118.** What does the Manual Approval stage do?

**Answer:** A human approves in the console or CLI before Deploy-Prod

It sits between Deploy-Staging and Deploy-Prod.

**Q119.** What does least privilege mean for the CodeBuild role here?

**Answer:** It can push to ECR, write logs and read source, but cannot call ecs:UpdateService

Each role gets only what its stage needs.

**Q120.** How is production deployed in the pipeline?

**Answer:** Blue/green via CodeDeploy to ECS

Deploy-Prod targets the prod cluster.

## Level 4 · Master

### [AWS Well-Architected Framework](level-4/01-well-architected-framework.md)

**Q121.** How many pillars does the Well-Architected Framework have?

**Answer:** Six

Operational Excellence, Security, Reliability, Performance Efficiency, Cost Optimization and Sustainability.

**Q122.** Which pillar asks 'Are you spending money only where it delivers value?'

**Answer:** Cost Optimization

Each pillar has a core question.

**Q123.** What does a Well-Architected review make you do about trade-offs?

**Answer:** State them deliberately rather than by accident

Higher reliability usually costs more.

**Q124.** What do unanswered questions surface as?

**Answer:** Risks, flagged as High or Medium Risk Issues

The lens review report summarizes high and medium risks per pillar.

### [Advanced Serverless (EventBridge at Scale)](level-4/02-advanced-serverless-eventbridge.md)

**Q125.** Why create custom event buses per domain?

**Answer:** So application events don't mix with AWS service events, and cross-account access can be scoped per bus

The default bus receives AWS service events.

**Q126.** What does put-permission allow?

**Answer:** Another AWS account to publish to your bus without sharing credentials

The shipping account uses its own IAM role.

**Q127.** What does content-based pattern matching on detail.tier achieve?

**Answer:** priorityFulfillment fires only for enterprise orders with no tier check in code

The filtering happens at the bus level.

**Q128.** What does the schema registry offer?

**Answer:** Inferred schemas and generated typed code bindings, turning field knowledge from tribal to explicit

For example OrderPlaced schemas for consumers.

### [Data Lakes &amp; Analytics (S3, Athena, Glue, Redshift)](level-4/03-data-lakes-analytics.md)

**Q129.** What is the data lake pattern in this lesson?

**Answer:** Raw data in S3, described by the Glue Data Catalog and queried in place by Athena

No data movement and no cluster to manage for ad-hoc queries.

**Q130.** Why partition S3 data by folders like year=2026/month=08/?

**Answer:** Query engines skip irrelevant partitions, which cuts cost since Athena bills per byte scanned

Columnar compressed formats such as Parquet help too.

**Q131.** What does a Glue crawler do?

**Answer:** Infers schema and partitions from the S3 layout and registers a table in the Data Catalog

Athena, Redshift Spectrum and EMR can all use the same catalog entry.

**Q132.** How is Athena billed?

**Answer:** Per query, by bytes scanned

It has no persistent infrastructure to provision.

### [Kubernetes at Scale (EKS Production Patterns)](level-4/04-eks-at-scale.md)

**Q133.** How does Karpenter differ from Cluster Autoscaler?

**Answer:** It provisions right-sized nodes directly for pending pods without ASGs, within seconds

It can mix spot and on-demand and consolidates underutilized nodes.

**Q134.** What does a Karpenter NodePool define?

**Answer:** What kinds of nodes it's allowed to launch, e.g. instance categories and capacity types, with limits

The example allows c, m and r categories on spot or on-demand.

**Q135.** What does GitOps with Argo CD mean?

**Answer:** The cluster continuously reconciles from a Git repo instead of kubectl apply from a laptop

The Application points at a repoURL, revision and path.

**Q136.** What do managed node groups do that self-managed ones don't?

**Answer:** Handle draining and rolling updates for you

Self-managed gives full control but requires reimplementing that lifecycle.

### [Multi-Account Strategy (Organizations, Control Tower)](level-4/05-multi-account-strategy.md)

**Q137.** Why use multiple AWS accounts?

**Answer:** Hard blast-radius boundaries: billing, quotas and IAM are already account-scoped

One team's mistake can't affect another's production.

**Q138.** What does Control Tower automate?

**Answer:** The landing zone: log archive and audit accounts, baseline SCPs and an Account Factory for compliant new accounts

New accounts are compliant from creation.

**Q139.** What is the difference between detective and preventive guardrails?

**Answer:** Detective flags non-compliant resources via Config rules; preventive blocks actions outright via SCPs

Control Tower enforces both.

**Q140.** Which is a typical OU?

**Answer:** Security, Infrastructure, Workloads/Prod, Workloads/NonProd, Sandbox

Each OU carries its own SCPs.

### [Cost Optimization at Scale](level-4/06-cost-optimization-at-scale.md)

**Q141.** Which commitment is the most flexible?

**Answer:** Compute Savings Plans, applying across EC2, Fargate and Lambda

They give a slightly lower discount than instance-family-locked options.

**Q142.** What do Spot Instances offer and risk?

**Answer:** The largest discount, but they can be reclaimed

Use them for interruption-tolerant workloads.

**Q143.** What does capacityOptimized allocation pick?

**Answer:** Spot pools with the deepest available capacity (lowest interruption risk) rather than the cheapest price

Mixing several instance types widens the pool.

**Q144.** What does get-savings-plans-purchase-recommendation do?

**Answer:** Analyzes recent usage and recommends a commitment level that covers steady state without over-committing

It looks back 30 or 60 days.

### [Advanced Networking (Direct Connect, Global Accelerator)](level-4/07-advanced-networking.md)

**Q145.** How does Direct Connect differ from a VPN?

**Answer:** A dedicated physical connection that bypasses the public internet; a VPN still traverses it

VPN has variable latency and shared bandwidth.

**Q146.** Why does Direct Connect provisioning take days to weeks?

**Answer:** It is physical: a provider uses AWS's Letter of Authorization to cross-connect at the location

Unlike every other resource in the path.

**Q147.** What does a public VIF reach?

**Answer:** AWS public service endpoints such as S3 and DynamoDB over the dedicated link

A private VIF connects to a VPC; a transit VIF to a Direct Connect gateway.

**Q148.** What is the production practice for Direct Connect resiliency?

**Answer:** Keep a Site-to-Site VPN as backup, failing over via BGP route priority

The highest resiliency is separate connections at two locations with diverse providers.

### [Compliance &amp; Governance (Config, Audit Manager)](level-4/08-compliance-governance.md)

**Q149.** What does AWS Config do?

**Answer:** Records every resource's configuration and changes and evaluates them against rules

A NON_COMPLIANT finding shows which resource drifted and when.

**Q150.** How can Config rules be evaluated?

**Answer:** On change (near-real-time) or on a periodic schedule

Unlike a point-in-time manual audit.

**Q151.** What is the lesson's advice on auto-remediation?

**Answer:** Good for well-understood low-risk fixes; test with Automatic: false first and review before auto-apply

It could disrupt legitimate configurations.

**Q152.** How does compliance differ from threat detection?

**Answer:** Compliance verifies resources stay configured as policy requires and produces audit evidence

Examples: encryption enabled, no public S3 buckets.

### [Chaos Engineering &amp; Resilience Testing](level-4/09-chaos-engineering-resilience.md)

**Q153.** What is chaos engineering?

**Answer:** Deliberately injecting failures in a controlled way to see if the design survives them

It's better than finding out in a real outage.

**Q154.** What makes FIS experiments safe on real infrastructure?

**Answer:** stopConditions, such as a CloudWatch alarm that halts the experiment

FIS halts if the high-error-rate alarm fires.

**Q155.** Which action simulates a network partition between AZs?

**Answer:** aws:network:disrupt-connectivity

It directly tests multi-AZ reliability claims.

**Q156.** What does PERCENT(50) selectionMode do in the template?

**Answer:** Targets half of the matching tagged instances

The resourceTags select staging training-app instances.

### [Capstone Project — Production-Grade Cloud Architecture](level-4/10-capstone-project.md)

**Q157.** What goes in the log-archive account?

**Answer:** Org CloudTrail and the Config aggregator

It sits in the Security OU.

**Q158.** How is the Transit Gateway shared to workload accounts?

**Answer:** Via RAM (Resource Access Manager) to the Workloads OU

The network account owns the TGW.

**Q159.** Why does remote Terraform state with DynamoDB locking live in a shared account bucket?

**Answer:** Both staging and prod pipelines share the same backend safely

It prevents concurrent applies from corrupting state.

**Q160.** What do the OU-level SCPs do in the capstone?

**Answer:** Restrict regions and block root-user API calls org-wide

New accounts come from Control Tower's Account Factory.

## Mixed questions across all levels

**Q161.** You must give a new teammate access to deploy but not to billing or IAM. What is the best IAM approach?

**Answer:** Add them to a group with a least-privilege policy

Groups carry policies for a team, and least privilege limits blast radius.

**Q162.** The root user's credentials leak. Why is that worse than a normal IAM user leak?

**Answer:** The root user cannot be restricted by any policy

It has unrestricted access to everything, including billing.

**Q163.** A Lambda function needs to read one S3 bucket. Where should that permission live?

**Answer:** On the function's execution role

The execution role, not the caller, determines what the function can access.

**Q164.** You want SSH to an EC2 instance only from your own IP. Which security group rule?

**Answer:** Inbound TCP 22 from your IP /32

Restricting the source reduces who can even attempt to connect.

**Q165.** Your RDS database must accept connections only from your app servers. What is the standard pattern?

**Answer:** Allow port 5432 from the app's security group (--source-group)

Source-group rules cover every current and future member of the app tier.

**Q166.** A private-subnet instance must download updates but never be reachable from the internet. What do you add?

**Answer:** A NAT Gateway

NAT lets private resources make outbound calls only.

**Q167.** Your static site on S3 needs managed HTTPS, global caching and a bucket that stays private. What do you use?

**Answer:** CloudFront with Origin Access Control

OAC lets CloudFront read a private bucket.

**Q168.** Traffic to your web tier spikes at night, and one instance is a single point of failure. What combination fixes both?

**Answer:** An Auto Scaling Group behind an Application Load Balancer

The ASG replaces failed instances and scales; the ALB spreads traffic and health-checks targets.

**Q169.** Orders must be processed by one of many workers, and failed messages must not block the queue forever. What do you use?

**Answer:** SQS with a dead-letter queue

SQS delivers each message to one consumer; the DLQ catches repeated failures.

**Q170.** One order event must trigger email, inventory and analytics independently. Which pattern?

**Answer:** SNS fan-out to multiple SQS queues

SNS copies the message to every subscriber.

**Q171.** A DynamoDB table must return all orders for one customer in one efficient query. How do you model the key?

**Answer:** Partition key customerId and sort key orderId

Items sharing a partition key are stored together and range-queryable.

**Q172.** A password for your database must be rotated automatically every 30 days. Which service?

**Answer:** Secrets Manager with rotation

Secrets Manager supports automatic rotation via a Lambda function.

**Q173.** You want to be warned before your monthly spend passes 80% of $50, but a budget alone won't stop resources. What does it do?

**Answer:** Notifies you; stopping resources needs an extra automated action

A budget only notifies unless wired to an action.

**Q174.** You describe your infrastructure in a template so it can be recreated and torn down with one command. What is this called and which native service does it?

**Answer:** Infrastructure as Code with CloudFormation

The stack is created and deleted as a unit.

**Q175.** Several teams share one AWS account and one team's mistake took down another's production. What is the structural fix?

**Answer:** Separate accounts under Organizations

Accounts give hard blast-radius boundaries for billing, quotas and IAM.

**Q176.** You want to stop anyone in an OU from using regions outside us-east-1 and us-west-2 even if they are administrators. What do you use?

**Answer:** A Service Control Policy

SCPs set a ceiling for accounts and OUs.

**Q177.** A request passes through API Gateway, Lambda, DynamoDB and SQS and is slow somewhere. Which service shows where?

**Answer:** X-Ray tracing

One trace ID follows the request across services.

**Q178.** Your RTO is minutes and your RPO is seconds, and you have a moderate budget. Which DR strategy fits best?

**Answer:** Warm standby

Backup and restore takes hours; warm standby recovers in minutes with seconds-minutes of data loss.

**Q179.** Ad-hoc queries over years of S3 data should only be billed for what they scan. Which pairing fits?

**Answer:** Athena with a partitioned columnar format like Parquet

Athena bills per byte scanned; partitions and columnar formats cut it.

**Q180.** Your pipeline builds a Docker image per commit. How should it tag images so you can roll back to a specific build?

**Answer:** With the commit SHA

A SHA tag identifies exactly which build is running.

---

More free practice: the [AWS Mastery Path](index.md) curriculum, the [Interview Prep Pack](https://learnmasterypath.com/interview-pack/) and the [free resume ATS check](https://learnmasterypath.com/resume/).
