# Terraform on AWS

Every prior module used CloudFormation or the CLI directly. **Terraform**
is HashiCorp's open-source IaC tool: it manages AWS (and any other
provider) resources using its own declarative language, HCL, and its
own state model — worth learning because it's the most widely used IaC
tool outside AWS shops, and it manages multi-cloud/non-AWS resources
CloudFormation can't touch.

## Install and configure

```bash
terraform version
# Terraform v1.9.0

# Credentials come from the same sources the AWS CLI uses:
# environment variables, ~/.aws/credentials, or an assumed role.
export AWS_PROFILE=training
```

## Provider and a first resource

```hcl
# main.tf
terraform {
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.0" }
  }
}

provider "aws" {
  region = "us-east-1"
}

resource "aws_s3_bucket" "training" {
  bucket = "training-app-bucket-8842"

  tags = {
    Environment = "dev"
    ManagedBy   = "terraform"
  }
}
```

```bash
terraform init
# Initializing the backend...
# Terraform has been successfully initialized!

terraform plan
# Terraform will perform the following actions:
#   + aws_s3_bucket.training will be created
# Plan: 1 to add, 0 to change, 0 to destroy.

terraform apply
# Do you want to perform these actions? Enter a value: yes
# aws_s3_bucket.training: Creating...
# Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

`terraform plan` is Terraform's equivalent of CloudFormation change
sets — always read it before `apply`, especially on shared state.

## CloudFormation vs. Terraform

| | CloudFormation | Terraform |
|---|---|---|
| Scope | AWS only | Any provider (AWS, GCP, Azure, Kubernetes, ...) |
| Language | JSON/YAML | HCL |
| State | Managed by AWS, invisible to you | Explicit `.tfstate` file you own |
| Drift detection | `detect-stack-drift` | `terraform plan` (implicit, every run) |
| Rollback on failure | Automatic | Manual (no built-in rollback) |
| Modules/reuse | Nested stacks | First-class `module` blocks |

The state file is the biggest conceptual shift: Terraform must own an
up-to-date record of every resource's real-world attributes to compute
diffs. Losing or corrupting it is the most common way to end up with
"Terraform thinks a resource doesn't exist but AWS says it does."

## Remote state and locking

Local state (a `terraform.tfstate` file on disk) breaks the moment two
people run `apply` at once. Production setups store state in S3 with
DynamoDB for locking:

```hcl
terraform {
  backend "s3" {
    bucket         = "training-tfstate-bucket"
    key            = "training-app/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```

```bash
terraform init -migrate-state
```

The DynamoDB table needs a `LockID` string partition key; Terraform
writes a lock item there for the duration of any `plan`/`apply`, so a
second concurrent run blocks instead of corrupting state.

## Variables, outputs, and modules

```hcl
# variables.tf
variable "environment" {
  type    = string
  default = "dev"
}

# outputs.tf
output "bucket_arn" {
  value = aws_s3_bucket.training.arn
}
```

```bash
terraform apply -var="environment=staging"
terraform output bucket_arn
# "arn:aws:s3:::training-app-bucket-8842"
```

A `module` block wraps a reusable set of resources (e.g., a standard
VPC layout) so multiple environments call the same module with
different variables instead of copy-pasting HCL.

## Gotchas

- **State drift**: if someone changes a resource in the console (or via
  CLI) instead of through Terraform, the next `plan` shows a diff that
  tries to revert it — `terraform apply -refresh-only` reconciles state
  to reality without changing infrastructure, but doesn't fix the root
  habit.
- **`.tfstate` contains resource attributes in plaintext**, including
  some secrets (e.g., RDS master passwords set via variables) — never
  commit it to git; the S3 backend should have versioning and
  encryption enabled, not public access.
- **`terraform destroy` has no confirmation beyond a yes/no prompt** and
  no automatic rollback — always run `terraform plan -destroy` first to
  see exactly what it's about to remove.
- **Provider version pinning matters** — an unpinned `aws` provider can
  pick up a new major version with breaking resource schema changes
  between runs on different machines.
- **Import is manual**: unlike CloudFormation's drift detection,
  bringing an existing hand-created resource under Terraform requires
  `terraform import <address> <id>` plus writing matching HCL yourself.

## Cheat sheet

| Command | Purpose |
|---|---|
| `terraform init` | Download providers, configure backend |
| `terraform plan` | Preview changes |
| `terraform apply` | Apply changes |
| `terraform destroy` | Tear down everything in state |
| `terraform state list` | List resources Terraform is tracking |
| `terraform import <addr> <id>` | Bring an existing resource under management |
| `terraform fmt` / `validate` | Format / syntax-check HCL |

## How It Actually Works

Terraform's core mechanism is a **declarative diff-and-apply engine** built
on a dependency graph, conceptually similar to CloudFormation but
implemented entirely client-side (or in your chosen remote runner) rather
than as an AWS-native control-plane feature. `terraform plan` builds this
graph from your `.tf` files (edges inferred from resource references), then
compares each resource's desired attributes against the last-known state
recorded in the **state file** — not against AWS's live API each time for
every attribute; it uses the state file as its source of truth for "what did
I create," refreshing it against real AWS resources first specifically to
detect drift.

The **provider** is what actually talks to AWS: it's a separate binary
implementing Terraform's plugin protocol, translating each resource block
into the equivalent AWS API calls (via the same underlying SDK the CLI
uses), and mapping API responses back into the schema Terraform tracks in
state — this indirection is why a manual Console change to a
Terraform-managed resource causes drift: Terraform's state file still
records the old values, and the next `plan` will propose "fixing" it back,
because Terraform has no mechanism to detect out-of-band changes except by
diffing against its own recorded state.

State file locking (typically via DynamoDB when using an S3 backend) exists
because concurrent `apply` runs manipulating the same state file
non-atomically could corrupt it or produce conflicting operations against
the same real resources — the lock is a simple conditional-write mutex in
DynamoDB, acquired before Terraform reads state and released after the
operation completes, which is exactly the same coordination problem
distributed databases solve with compare-and-swap, applied to
infrastructure tooling.

## Exercise

Write a Terraform configuration that creates an S3 bucket and a DynamoDB
table (for a future state backend), using variables for the bucket name
and table name and an output for each resource's ARN. Run `terraform
plan` and confirm it reports "2 to add, 0 to change, 0 to destroy"
before ever running `apply`.
