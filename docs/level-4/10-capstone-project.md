# Capstone Project — Production-Grade Cloud Architecture

This capstone pulls together the full path: multi-account structure
(module 5), IaC (Level 3 module 5), CI/CD (Level 3 module 6),
observability (Level 3 module 8), and cost controls (module 6) into one
reference architecture for a production SaaS-style application.

## Architecture overview

```
Organization (Control Tower landing zone)
├── Security OU
│   └── log-archive account       ← org CloudTrail, Config aggregator
├── Infrastructure OU
│   └── network account           ← Transit Gateway, Direct Connect, shared via RAM
├── Workloads/Prod OU
│   └── prod account               ← EKS cluster, RDS Multi-AZ, prod ECR images
└── Workloads/NonProd OU
    └── staging account            ← same stack, smaller scale
```

Each account is provisioned via Control Tower's Account Factory
(module 5), with SCPs at the OU level restricting region and blocking
root-user API calls org-wide, and an org-wide CloudTrail trail landing
in the log-archive account.

## Step 1 — network foundation

```bash
# In the network account: TGW as the hub (Level 3 module 1)
aws ec2 create-transit-gateway --description "org-hub" \
  --options AmazonSideAsn=64512,DefaultRouteTableAssociation=enable

# Share it to the Workloads OU via RAM (module 5)
aws ram create-resource-share \
  --name shared-tgw \
  --resource-arns arn:aws:ec2:us-east-1:111111111111:transit-gateway/tgw-0abc123 \
  --principals ou-abcd-workloads1
```

## Step 2 — infrastructure as code (Terraform)

```hcl
# environments/prod/main.tf
module "vpc" {
  source   = "../../modules/vpc"
  vpc_cidr = "10.10.0.0/16"
  tgw_id   = "tgw-0abc123"
}

module "eks" {
  source          = "../../modules/eks"
  cluster_name    = "prod-cluster"
  vpc_id          = module.vpc.vpc_id
  subnet_ids      = module.vpc.private_subnet_ids
  node_capacity_type = "ON_DEMAND"
}

module "rds" {
  source            = "../../modules/rds"
  identifier        = "prod-db"
  multi_az          = true
  backup_retention  = 7
  instance_class    = "db.r6g.large"
}
```

```bash
terraform init -backend-config="key=prod/terraform.tfstate"
terraform plan -out=prod.tfplan
terraform apply prod.tfplan
```

Remote state (Level 3 module 5) lives in the network or a dedicated
tooling account's S3 bucket with DynamoDB locking, so both staging and
prod pipelines share the same backend safely.

## Step 3 — CI/CD pipeline per environment

Extending the Level 3 capstone pipeline: source triggers a build,
staging deploys automatically, and production requires manual approval
plus a blue/green ECS/EKS rollout with automatic rollback on alarm —
identical mechanism to Level 3 module 10, now targeting a separate AWS
account per environment via cross-account role assumption
(`sts:AssumeRole`, Level 3 module 4) rather than a separate cluster in
one account.

```bash
aws sts assume-role \
  --role-arn arn:aws:iam::444444444444:role/CrossAccountDeployRole \
  --role-session-name pipeline-deploy \
  --external-id shared-secret-456
```

## Step 4 — observability

```bash
# X-Ray tracing across the EKS-hosted API and its Lambda-based async workers
aws lambda update-function-configuration --function-name order-worker --tracing-config Mode=Active

# CloudWatch alarm feeding both CodeDeploy auto-rollback and FIS stop conditions
aws cloudwatch put-metric-alarm \
  --alarm-name prod-high-error-rate \
  --metric-name 5XXError --namespace AWS/ApplicationELB \
  --statistic Sum --period 60 --threshold 50 --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2
```

The same `prod-high-error-rate` alarm serves double duty: it's the
rollback trigger for CodeDeploy deployments (Level 3 module 10) and the
stop condition for any FIS chaos experiment (module 9) run against
this environment.

## Step 5 — cost and compliance guardrails

```bash
# Compute Savings Plan sized from 60 days of usage across the org
aws ce get-savings-plans-purchase-recommendation \
  --savings-plans-type COMPUTE_SP --term-in-years ONE_YEAR \
  --payment-option NO_UPFRONT --lookback-period-in-days SIXTY_DAYS

# Conformance pack applied org-wide from the Security OU
aws configservice put-conformance-pack \
  --conformance-pack-name operational-best-practices \
  --template-s3-uri s3://aws-configservice-conformancepack-templates/Operational-Best-Practices-for-Encryption-and-Keys-Management.yaml
```

## Step 6 — resilience validation

Run an FIS experiment (module 9) against staging first — terminate one
EKS node — confirming Karpenter (module 4) provisions a replacement
and the ALB continues serving traffic within the SLO defined by the
Well-Architected review (module 1). Only after staging passes does the
same experiment template get scheduled against prod, with a narrower
blast radius and off-peak timing.

## Putting it together: request path

```
User → Global Accelerator (module 7) → ALB (prod account)
     → EKS pods (module 4, Karpenter-scaled)
     → RDS Multi-AZ (prod account) / EventBridge (module 2) for async work
     → CloudWatch + X-Ray (Level 3 module 8) for observability
     → CodePipeline cross-account deploy (Level 3 module 6 + module 5) for releases
```

## Gotchas

- **Cross-account IAM trust must be established before the pipeline
  ever runs** — a `CrossAccountDeployRole` with a trust policy missing
  the pipeline account's principal fails at `AssumeRole` with an opaque
  `AccessDenied`, not a helpful pipeline error.
- **Terraform state per environment must be genuinely isolated** — a
  shared state file across staging and prod risks one `terraform
  apply` accidentally modifying the wrong environment; separate state
  keys (or separate backends per account) are non-negotiable at this
  scale.
- **SCPs at the OU level apply before any account-level IAM policy is
  even evaluated** — a perfectly correct deploy role can still fail if
  an SCP on `Workloads/Prod` blocks the specific API call; debug
  `AccessDenied` errors by checking SCPs first in a multi-account
  setup, not last.
- **Chaos experiments and cost optimization can conflict** — an FIS
  experiment that terminates spot instances to test resilience may
  interact unpredictably with a Karpenter consolidation event
  happening for cost reasons at the same time; avoid scheduling both
  simultaneously.
- **This entire architecture was designed and syntax-verified, not
  provisioned** — no real accounts, clusters, or databases were created
  while writing this module, since doing so requires real credentials
  and incurs real cost; treat every command here as a verified-syntax
  starting point, not a tested deployment.

## How It Actually Works

This capstone's full request and deployment path composes nearly every
mechanism covered across all four levels into one system, and its
end-to-end behavior only makes sense once you trace each boundary
individually. A request still resolves through Route 53's in-infrastructure
alias lookup, lands at a CloudFront edge PoP or Global Accelerator Anycast
IP, and reaches compute (EKS pods behind an ALB, or Fargate tasks) whose
actual placement was decided by the Kubernetes scheduler or ECS's
bin-packing algorithm against real-time reported capacity — none of these
layers share a single control plane; each is independently reconciling its
own piece of desired state against observed reality, which is why a
capstone-scale outage investigation means tracing distinct
failure/recovery mechanisms at each boundary rather than looking for one
root cause.

Multi-account guardrails (SCPs intersecting with IAM, Config conformance
packs re-evaluating on every CloudTrail-captured change) and chaos-tested
resilience (FIS-injected faults exercising the same Multi-AZ/ASG/Route 53
failover logic modules earlier in this course covered in isolation) are
what make this capstone a genuine test of whether those individual
mechanisms compose correctly under real, coordinated failure — a guardrail
that works fine tested alone can still interact badly with an
auto-remediation Config rule or a Terraform apply running concurrently,
exactly the kind of emergent, cross-system behavior that only shows up when
every mechanism from this course is running together, under load, at once.

The billing and cost-optimization layer (Savings Plans rating, Spot
capacity reclamation, Intelligent-Tiering's access-pattern monitoring) is
likewise still operating asynchronously underneath this whole system the
entire time — none of it observable in real time from inside the
architecture itself, only after the fact once usage records propagate
through AWS's metering pipeline, which is exactly why a capstone's
cost-guardrail step has to rely on Budgets/Config as detective controls
layered on top, not a live cost readout wired into the request path.

## Stretch goals

- Add a second region to the prod account and extend the FIS game day
  to test a full regional failover using the DR patterns from Level 3
  module 7 and Global Accelerator's endpoint-group failover.
- Wire Audit Manager (module 8) to produce a quarterly evidence report
  scoped to this exact architecture, mapped to a real framework (e.g.,
  SOC 2) rather than the generic conformance pack used here.
- Replace the fixed `db.r6g.large` RDS instance with Aurora Global
  Database and update the DR/multi-region plan to use active-active
  writes instead of promote-on-failover.
- Add a fourth account (`Sandbox`) with a tightly scoped SCP and a
  short-lived resource budget, and automate its cleanup with a
  scheduled Lambda that terminates anything older than 48 hours.
