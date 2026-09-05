# 07 · Secrets Manager & Parameter Store

The Level 1 Lambda module set configuration with plain environment
variables (`STAGE=training`) — fine for non-sensitive config, but a
database password or API key sitting in plaintext function configuration
is visible to anyone with read access to the function, and never rotates.
This module covers AWS's two purpose-built stores for this: **Secrets
Manager** (secrets with built-in rotation) and **Systems Manager Parameter
Store** (general configuration, with an optional encrypted tier) — and
when to reach for each.

## Core concepts

| Concept | What it is |
|---|---|
| **Secret (Secrets Manager)** | A versioned, encrypted value with optional automatic rotation via a Lambda function. |
| **Parameter (SSM)** | A named config value; type `String`/`StringList` (plain) or `SecureString` (KMS-encrypted). |
| **Parameter hierarchy** | Path-like naming (`/training/app/db-host`) that lets you fetch a whole config tree at once. |
| **KMS key** | The encryption key backing `SecureString` parameters and all Secrets Manager values. |
| **Rotation** | Automatically generating a new secret value on a schedule and updating both the store and the target system (e.g. RDS). |

## Secrets Manager: store and read a secret

```bash
aws secretsmanager create-secret \
  --name training/db-credentials \
  --secret-string '{"username": "appuser", "password": "Tr@ining-Pass-2026"}'
# ARN: arn:aws:secretsmanager:us-east-1:123456789012:secret:training/db-credentials-AbCdEf

aws secretsmanager get-secret-value \
  --secret-id training/db-credentials \
  --query "SecretString" --output text
# {"username": "appuser", "password": "Tr@ining-Pass-2026"}
```

Every write creates a new **version** rather than overwriting in place —
`get-secret-value` defaults to the `AWSCURRENT` version, but prior
versions remain retrievable (useful mid-rotation, or to confirm what an
application actually read at a point in time).

## Updating a secret's value

```bash
aws secretsmanager put-secret-value \
  --secret-id training/db-credentials \
  --secret-string '{"username": "appuser", "password": "New-Rotated-Pass-99"}'
```

## Enabling automatic rotation

```bash
aws secretsmanager rotate-secret \
  --secret-id training/db-credentials \
  --rotation-lambda-arn arn:aws:lambda:us-east-1:123456789012:function:training-rotate-db-secret \
  --rotation-rules AutomaticallyAfterDays=30
```

AWS publishes ready-made rotation Lambda templates for RDS (MySQL,
PostgreSQL, etc.) via the Secrets Manager console/SAM app; a custom
rotation function follows a 4-step contract (`createSecret`,
`setSecret`, `testSecret`, `finishSecret`) so the old credential keeps
working until the new one is confirmed live — avoiding an outage mid
-rotation. If the database lives in a VPC, the rotation function needs
network access to it (a VPC-attached Lambda, module 1's networking
concerns apply here too).

## Parameter Store: plain and encrypted values

```bash
aws ssm put-parameter \
  --name /training/app/log-level \
  --value "info" --type String

aws ssm put-parameter \
  --name /training/app/db-password \
  --value "Tr@ining-Pass-2026" --type SecureString

aws ssm get-parameter --name /training/app/log-level
# Parameter.Value: info  (plaintext value returned directly)

aws ssm get-parameter --name /training/app/db-password
# Parameter.Value: AQICAHh...  (still ciphertext without --with-decryption)

aws ssm get-parameter --name /training/app/db-password --with-decryption
# Parameter.Value: Tr@ining-Pass-2026
```

`SecureString` parameters are encrypted with a KMS key (the AWS-managed
`alias/aws/ssm` by default, or a customer-managed key) — the caller needs
`kms:Decrypt` permission on that key **in addition to** `ssm:GetParameter`
to read the plaintext; missing either produces an access-denied error even
if the other permission looks fine.

## Fetching a whole config tree

```bash
aws ssm put-parameter --name /training/app/db-host --value "db.internal" --type String
aws ssm put-parameter --name /training/app/db-port --value "5432" --type String

aws ssm get-parameters-by-path \
  --path /training/app/ --recursive --with-decryption \
  --query "Parameters[].[Name,Value]" --output table
```

This is the pattern an application typically uses at startup: one call
loads every config value under its namespace, `SecureString` and plain
`String` alike, instead of one `get-parameter` call per value.

## Secrets Manager vs. Parameter Store

| | Secrets Manager | Parameter Store (Standard tier) |
|---|---|---|
| Encryption | Always encrypted (KMS) | Optional (`SecureString` vs. `String`) |
| Automatic rotation | Built-in, with Lambda | Not built-in (roll your own via EventBridge + Lambda) |
| Cost | Per secret + per API call beyond a small free tier | Free for Standard-tier parameters, generous API call allowance |
| Best for | Database credentials, API keys needing rotation | App config, feature flags, non-rotating values |

A common pattern: put a database password in Secrets Manager (for
rotation), and everything else — hostnames, log levels, feature flags —
in Parameter Store (free, no rotation needed).

## Referencing a secret from a task/function, not hardcoding it

```bash
# In an ECS task definition (module 1), reference the secret instead of a plaintext env var
```
```json
{
  "containerDefinitions": [{
    "name": "training-app",
    "secrets": [
      { "name": "DB_PASSWORD", "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789012:secret:training/db-credentials-AbCdEf" }
    ]
  }]
}
```

ECS resolves `secrets` entries at task launch time and injects them as
environment variables inside the container — the value never appears in
the task definition itself, only its ARN, so `describe-task-definition`
output stays safe to share.

!!! warning "Never fall back to plain Lambda/ECS environment variables for real secrets"
    Module 1 (Lambda) and this course's earlier examples used plain
    environment variables for non-sensitive config like `STAGE`. Anyone
    with `lambda:GetFunctionConfiguration` or `ecs:DescribeTaskDefinition`
    permission can read those values in plaintext — fine for a log level,
    never for a password or API key.

## Cheat sheet

| Command | Purpose |
|---|---|
| `aws secretsmanager create-secret --secret-string JSON` | Store a new secret value. |
| `aws secretsmanager get-secret-value --secret-id ID` | Retrieve the current secret value. |
| `aws secretsmanager put-secret-value` | Write a new version of a secret. |
| `aws secretsmanager rotate-secret --rotation-lambda-arn ARN --rotation-rules ...` | Enable scheduled automatic rotation. |
| `aws ssm put-parameter --type SecureString` | Store an encrypted config value. |
| `aws ssm get-parameter --with-decryption` | Read a `SecureString` parameter's plaintext. |
| `aws ssm get-parameters-by-path --recursive` | Fetch an entire config namespace at once. |

## How It Actually Works

Both services store data encrypted at rest via **AWS KMS envelope
encryption**: rather than encrypting your secret directly with your KMS key
(which would require a network call to KMS on every read), the service
generates a random per-secret **data key**, encrypts your secret with it
locally, then asks KMS to encrypt *that* data key with your chosen KMS key
and stores the encrypted data key alongside the ciphertext. Decryption
reverses this: on read, the service sends the encrypted data key to KMS to
be decrypted (this call is what actually gets logged in CloudTrail and
billed), then uses the returned plaintext data key locally to decrypt your
secret — your actual secret value is never transmitted to KMS at all, only
its wrapper key.

Secrets Manager's headline feature, **automatic rotation**, works by
invoking a Lambda function you (or AWS) provide on a schedule; that function
implements a four-step protocol (`createSecret`, `setSecret`, `testSecret`,
`finishSecret`) against the target service (e.g. RDS) so that a new
credential is generated, applied to the database, verified to work, and
only then promoted to "current" — the staged design exists specifically so
a partial failure mid-rotation doesn't lock out every client using the old
credential before the new one is confirmed working.

Parameter Store is architecturally simpler and cheaper because it skips
this workflow entirely — a `SecureString` parameter is just KMS envelope
encryption with no built-in rotation orchestration, which is exactly the
trade-off that makes it fine for static configuration but a poor fit for
credentials that need to rotate on a schedule without human involvement.

## Exercise

1. Store a fake database credential pair as a Secrets Manager secret, and
   read it back with `get-secret-value`.
2. Write a new version with `put-secret-value`, then confirm you can still
   retrieve the previous version by its version ID (`--version-id`, found
   via `list-secret-version-ids`).
3. Store 3 config values under `/training/app/` in Parameter Store — one
   plain `String`, two `SecureString` — and fetch all 3 in one call with
   `get-parameters-by-path --with-decryption`.
4. Try `get-parameter --with-decryption` on a `SecureString` using an IAM
   principal that has `ssm:GetParameter` but not `kms:Decrypt` on the
   backing key, and confirm (and explain) the access-denied error.
5. Reference the Secrets Manager secret from an ECS task definition's
   `secrets` field (module 1) instead of a plaintext environment variable.
6. Delete the secret (`delete-secret`, note the default recovery window
   before permanent deletion) and the parameters when done.
