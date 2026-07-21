# 09 · CloudFormation & IaC Intro

Every module so far has provisioned resources by running CLI commands one
at a time — which works, but leaves no single record of what you built, no
easy way to reproduce it, and no clean way to tear it all down except by
remembering (or grepping your terminal history for) every resource ID.
**Infrastructure as Code (IaC)** solves this by describing your
infrastructure in a text file, checked into version control like any other
code. This module introduces CloudFormation, AWS's native IaC service, by
rebuilding a small piece of what you did by hand in earlier modules — as a
single template you can create and destroy with one command each.

## Why IaC matters

| Manual CLI/console changes | Infrastructure as Code |
|---|---|
| No record of what exists or why | The template *is* the record |
| Hard to reproduce in a second environment | Same template deploys to dev/stage/prod |
| Teardown means remembering every resource | One command deletes everything the template created |
| Drift (console click) is invisible | Diffing against the template reveals drift |
| No review process | Templates go through the same PR review as app code |

## Templates and stacks

A CloudFormation **template** is a YAML (or JSON) file describing resources.
A **stack** is a running instance of a template — a group of real AWS
resources CloudFormation created and tracks together, so it can update or
delete them as a unit.

```yaml
# s3-website-stack.yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: A single S3 bucket configured for public static website hosting.

Parameters:
  BucketName:
    Type: String
    Description: Globally-unique S3 bucket name.

Resources:
  WebsiteBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Ref BucketName
      AccessControl: PublicRead
      WebsiteConfiguration:
        IndexDocument: index.html
        ErrorDocument: error.html
      PublicAccessBlockConfiguration:
        BlockPublicAcls: false
        IgnorePublicAcls: false
        BlockPublicPolicy: false
        RestrictPublicBuckets: false

  WebsiteBucketPolicy:
    Type: AWS::S3::BucketPolicy
    Properties:
      Bucket: !Ref WebsiteBucket
      PolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Sid: PublicReadForWebsite
            Effect: Allow
            Principal: "*"
            Action: "s3:GetObject"
            Resource: !Sub "${WebsiteBucket.Arn}/*"

Outputs:
  WebsiteURL:
    Description: URL of the static website
    Value: !GetAtt WebsiteBucket.WebsiteURL
  BucketArn:
    Value: !GetAtt WebsiteBucket.Arn
```

This single file replaces the several separate `aws s3api` calls from Module
4 — the bucket, its website configuration, its public-access settings, and
its bucket policy, all declared together and created (or deleted) as one
unit.

## Key template sections

| Section | Purpose |
|---|---|
| `Parameters` | Inputs supplied at deploy time (like function arguments). |
| `Resources` | The actual AWS resources to create — the only required section. |
| `Outputs` | Values exposed after creation (ARNs, URLs) — readable by you or by other stacks. |
| `!Ref` | References a parameter's value or a resource's primary identifier. |
| `!GetAtt` | Gets a specific attribute of a resource (e.g. its ARN, its URL). |
| `!Sub` | String interpolation, substituting `${...}` references. |

## Deploy the stack

```bash
aws cloudformation deploy \
  --template-file s3-website-stack.yaml \
  --stack-name training-website-stack \
  --parameter-overrides BucketName=yourname-cfn-training-2026
```

`aws cloudformation deploy` is a convenience wrapper: it creates a
**change set** (a preview diff of what will change), applies it, and waits
for completion — all in one command. Watch progress in real time:

```bash
aws cloudformation describe-stack-events \
  --stack-name training-website-stack \
  --query "StackEvents[].[LogicalResourceId,ResourceStatus]" \
  --output table
```

Read the outputs once it's done:

```bash
aws cloudformation describe-stacks \
  --stack-name training-website-stack \
  --query "Stacks[0].Outputs"
# [
#   {"OutputKey": "WebsiteURL", "OutputValue": "http://yourname-cfn-training-2026.s3-website-us-east-1.amazonaws.com"},
#   {"OutputKey": "BucketArn", "OutputValue": "arn:aws:s3:::yourname-cfn-training-2026"}
# ]
```

## Updating a stack

Edit the template (e.g. add an `ErrorDocument` change, or a second
resource) and re-run the same `deploy` command — CloudFormation computes the
diff and only touches what changed:

```bash
# After editing s3-website-stack.yaml
aws cloudformation deploy \
  --template-file s3-website-stack.yaml \
  --stack-name training-website-stack \
  --parameter-overrides BucketName=yourname-cfn-training-2026
```

If you want to preview a change before committing to it, create the change
set manually without executing it:

```bash
aws cloudformation create-change-set \
  --stack-name training-website-stack \
  --template-body file://s3-website-stack.yaml \
  --change-set-name preview-1

aws cloudformation describe-change-set \
  --stack-name training-website-stack \
  --change-set-name preview-1 \
  --query "Changes[].ResourceChange.[Action,LogicalResourceId]"
```

## Deleting a stack (the entire point of IaC for cleanup)

```bash
aws cloudformation delete-stack --stack-name training-website-stack
aws cloudformation wait stack-delete-complete --stack-name training-website-stack
```

This removes **every resource the stack created** — no need to remember
individual bucket names, policy ARNs, or run several separate delete
commands. This is the single biggest practical win of IaC for a training
environment: one command undoes everything.

!!! note "Some resources need to be emptied before they can be deleted"
    S3 buckets containing objects will fail to delete via CloudFormation
    unless empty. For real projects, either add a Lambda-backed custom
    resource to empty the bucket automatically, or empty it manually
    (`aws s3 rm s3://BUCKET --recursive`) before deleting the stack.

## CloudFormation vs. Terraform (preview)

CloudFormation is AWS-native (no separate tool to install, deep integration
with `deploy`/rollback), but only manages AWS. **Terraform** (covered in
Level 3) is cloud-agnostic and has become the more common choice in
multi-cloud or platform-engineering teams. Both express the same core idea —
declare desired state, let the tool reconcile it — and learning
CloudFormation's mental model here transfers directly.

## Cheat sheet

| Command | Purpose |
|---|---|
| `aws cloudformation deploy --template-file F --stack-name N --parameter-overrides K=V` | Create or update a stack from a template. |
| `aws cloudformation describe-stacks --stack-name N` | View a stack's status and outputs. |
| `aws cloudformation describe-stack-events --stack-name N` | See per-resource creation/update progress. |
| `aws cloudformation create-change-set` / `describe-change-set` | Preview changes before applying. |
| `aws cloudformation delete-stack --stack-name N` | Delete every resource the stack owns. |
| `aws cloudformation wait stack-delete-complete` | Block until deletion finishes. |
| `!Ref`, `!GetAtt`, `!Sub` | Core template intrinsic functions. |

## Exercise

1. Write the `s3-website-stack.yaml` template above (or your own variant)
   and deploy it with a bucket name unique to you.
2. Upload an `index.html` and `error.html` to the created bucket (outside
   the template, via `aws s3 cp`, as in Module 4) and visit the
   `WebsiteURL` output to confirm it works.
3. Add a second resource to the template — e.g. a
   `AWS::CloudWatch::Alarm` or a second bucket — and re-deploy, watching
   `describe-stack-events` to confirm only the new resource is created (the
   existing bucket is left untouched).
4. Create a change set for one more edit without executing it, and read its
   `describe-change-set` output to confirm it shows exactly the change you
   expect before applying anything.
5. Empty the bucket (`aws s3 rm s3://BUCKET --recursive`), then delete the
   stack and confirm with `describe-stacks` that it's gone (or errors with
   "does not exist").
