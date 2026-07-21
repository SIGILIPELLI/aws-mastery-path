# 01 · Setup & AWS CLI

Every module in this course assumes you have an AWS account and a working
AWS CLI on your machine. This module walks through creating the account
safely (never using the root user day-to-day), installing the CLI, creating
an IAM user just for CLI access, and understanding the basic shape of every
CLI command you'll run from here on. Getting this right up front saves you
from two common beginner mistakes: doing everything as the root user (a
security risk), and not understanding regions/profiles (a source of very
confusing "where did my resource go?" bugs).

## Create your AWS account

1. Go to [aws.amazon.com](https://aws.amazon.com) and click **Create an AWS Account**.
2. Provide an email, password, and AWS account name.
3. Add a payment method (required even for free-tier usage) and complete
   identity verification (phone/SMS).
4. Choose the **Basic support plan** (free) unless you have a reason for more.

The email/password you just created is the **root user** — it has
unrestricted access to everything in the account, including billing. AWS's
own guidance, and this course's, is: **use the root user only to create your
first IAM user, then stop using it for daily work.**

!!! warning "Never use the root user for day-to-day work"
    The root user cannot be restricted by any policy. If its credentials
    leak, an attacker has full control of your account and your bill. Lock it
    down with MFA (covered in the next module) and set it aside.

## Enable an IAM user for yourself

Log in as root once, go to the **IAM** console, and create an administrative
IAM user for yourself:

1. IAM → Users → **Create user**.
2. Username: e.g. `yourname-admin`.
3. Attach the AWS-managed policy `AdministratorAccess` (broad on purpose —
   this is *your* day-to-day login, not a service credential; least-privilege
   for service roles is covered in the next module).
4. Under **Security credentials**, create an **access key** with use case
   "Command Line Interface (CLI)". Save the **Access Key ID** and **Secret
   Access Key** somewhere safe (a password manager) — the secret is shown
   only once.

From now on, log in to the console and run the CLI as this IAM user, not
root.

## Install the AWS CLI

```bash
# macOS (Homebrew)
brew install awscli

# Linux (x86_64)
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Windows: download and run the MSI installer from
# https://awscli.amazonaws.com/AWSCLIV2.msi
```

Verify the install:

```bash
aws --version
# aws-cli/2.x.x Python/3.x.x Darwin/23.x.x source/arm64
```

## Configure your credentials

```bash
aws configure
# AWS Access Key ID [None]: AKIAIOSFODNN7EXAMPLE
# AWS Secret Access Key [None]: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
# Default region name [None]: us-east-1
# Default output format [None]: json
```

This writes two files under `~/.aws/`:

```ini
# ~/.aws/credentials
[default]
aws_access_key_id = AKIAIOSFODNN7EXAMPLE
aws_secret_access_key = wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY

# ~/.aws/config
[default]
region = us-east-1
output = json
```

Confirm the CLI can authenticate as you:

```bash
aws sts get-caller-identity
# {
#     "UserId": "AIDACKCEVSQ6C2EXAMPLE",
#     "Account": "123456789012",
#     "Arn": "arn:aws:iam::123456789012:user/yourname-admin"
# }
```

If this returns your IAM user's ARN (not an error), the CLI is correctly
configured.

## Regions and why they matter

AWS resources live in a **region** (e.g. `us-east-1` = N. Virginia,
`eu-west-1` = Ireland, `ap-south-1` = Mumbai). Most services are
region-scoped: an EC2 instance you launch in `us-east-1` simply does not
exist when you list instances in `eu-west-1` — it's not hidden, it's a
different set of servers entirely. A few services are global (IAM, Route 53,
CloudFront) and don't have this distinction.

```bash
# List instances in the default region (from config)
aws ec2 describe-instances

# Override the region for a single command
aws ec2 describe-instances --region eu-west-1

# List every region available to your account
aws ec2 describe-regions --query "Regions[].RegionName" --output table
```

Pick one region for this whole course (e.g. `us-east-1`, which typically has
the earliest access to new features and the lowest prices) and stick with it
unless a module says otherwise — it avoids the "I can't find my S3 bucket"
confusion later.

## Profiles: managing more than one identity

**Profiles** let the CLI hold multiple named credential sets — useful once
you have a personal account and a work account, or separate admin vs.
read-only credentials.

```bash
# Add a second, named profile interactively
aws configure --profile training

# Use it for a single command
aws s3 ls --profile training

# Or export it for the whole shell session
export AWS_PROFILE=training
aws sts get-caller-identity   # now uses the "training" profile
```

```ini
# ~/.aws/credentials can hold several profiles side by side
[default]
aws_access_key_id = AKIA...DEFAULT
aws_secret_access_key = ...

[training]
aws_access_key_id = AKIA...TRAINING
aws_secret_access_key = ...
```

## Anatomy of a CLI command

Every AWS CLI command follows the same shape:

```bash
aws <service> <operation> [--parameters] [--region] [--profile] [--output]
```

```bash
# service = ec2, operation = describe-instances
aws ec2 describe-instances \
  --region us-east-1 \
  --profile training \
  --output table \
  --query "Reservations[].Instances[].[InstanceId,State.Name]"
```

`--query` runs a [JMESPath](https://jmespath.org/) expression against the
JSON response to filter it down to just what you need — indispensable once
responses get large (`describe-instances` returns a lot of nested detail per
instance).

## Cheat sheet

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

## Exercise

1. Create a second IAM user called `readonly-test` with the AWS-managed
   `ReadOnlyAccess` policy attached, and generate a CLI access key for it.
2. Configure it as a new CLI profile named `readonly`.
3. Run `aws sts get-caller-identity --profile readonly` to confirm it
   authenticates as the new user.
4. Run `aws ec2 describe-instances --profile readonly --output table` — it
   should succeed (empty list is fine, you haven't launched anything yet).
5. Try `aws ec2 run-instances --profile readonly ...` with any parameters —
   confirm it's denied with an `UnauthorizedOperation` error, since
   `ReadOnlyAccess` can view but not create resources. This is your first
   hands-on look at least-privilege access, covered fully in the next module.
