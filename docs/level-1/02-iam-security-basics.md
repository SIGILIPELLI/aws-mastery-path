# 02 · IAM & Security Basics

IAM (Identity and Access Management) is the service that controls *who* can
do *what* in your AWS account — it's global (not region-scoped) and it's the
single most important service to understand before you provision anything
else. Nearly every security incident in AWS traces back to an IAM mistake:
an overly broad policy, a leaked access key, or a missing MFA requirement.
This module covers the building blocks — users, groups, roles, policies —
and the two habits that prevent most incidents: least privilege and MFA.

## The four IAM building blocks

| Concept | What it is |
|---|---|
| **User** | An identity for a person or an application, with long-lived credentials (password and/or access keys). |
| **Group** | A named collection of users. Policies attached to the group apply to every user in it. |
| **Role** | An identity with *temporary* credentials, assumed by a user, application, or AWS service (e.g. an EC2 instance or Lambda function) — no long-lived secret to leak. |
| **Policy** | A JSON document listing allowed (or denied) actions on specific resources. Attached to users, groups, or roles. |

## Users and groups

Create a group and attach a policy to it, then add users to the group
instead of attaching policies to each user individually — this is how you
manage permissions for a team without repeating yourself:

```bash
aws iam create-group --group-name Developers

aws iam attach-group-policy \
  --group-name Developers \
  --policy-arn arn:aws:iam::aws:policy/PowerUserAccess

aws iam add-user-to-group \
  --group-name Developers \
  --user-name yourname-admin
```

List what a user can actually do (useful for debugging "access denied"
errors):

```bash
aws iam list-attached-user-policies --user-name yourname-admin
aws iam list-groups-for-user --user-name yourname-admin
```

## Policies: the anatomy of a permission

A policy is JSON with one or more **statements**, each an `Effect`
(`Allow`/`Deny`), one or more `Action`s, and one or more `Resource`s they
apply to:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ReadOwnBucket",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::my-training-bucket",
        "arn:aws:s3:::my-training-bucket/*"
      ]
    }
  ]
}
```

This grants read-only access to exactly one bucket — nothing else. Compare
that to `AmazonS3FullAccess`, which grants every S3 action on every bucket in
the account. **The principle of least privilege** means always preferring
the narrowest policy that lets the job get done, not the broadest one that's
convenient to write.

Create a custom policy and attach it to a group:

```bash
aws iam create-policy \
  --policy-name ReadOwnBucketPolicy \
  --policy-document file://read-own-bucket.json

aws iam attach-group-policy \
  --group-name Developers \
  --policy-arn arn:aws:iam::123456789012:policy/ReadOwnBucketPolicy
```

## Roles: permissions without long-lived secrets

A **role** has no password or permanent access key — instead, a trusted
principal (a user, an AWS service, another account) "assumes" it and
receives short-lived, auto-expiring credentials. This is why an EC2
instance or Lambda function should always run under a role, never under an
IAM user's access keys hardcoded into an app.

```bash
# Trust policy: who is allowed to assume this role
cat > ec2-trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "ec2.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

aws iam create-role \
  --role-name EC2-S3-ReadOnly \
  --assume-role-policy-document file://ec2-trust-policy.json

aws iam attach-role-policy \
  --role-name EC2-S3-ReadOnly \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# Create an "instance profile" (the wrapper EC2 uses to attach a role)
aws iam create-instance-profile --instance-profile-name EC2-S3-ReadOnly-Profile
aws iam add-role-to-instance-profile \
  --instance-profile-name EC2-S3-ReadOnly-Profile \
  --role-name EC2-S3-ReadOnly
```

You'll attach this instance profile to an EC2 instance in the next module —
the instance then reads S3 without any access key ever touching its disk.

## The principle of least privilege

Three habits that keep an account secure:

1. **Start narrow, widen only when something breaks.** Attach
   `ReadOnlyAccess` first; add specific write actions only once you hit a
   real `AccessDenied` error for something you actually need.
2. **Scope `Resource` to specific ARNs**, not `"Resource": "*"`, whenever the
   service supports it (S3, DynamoDB, and most others do).
3. **Prefer roles over long-lived access keys** wherever a workload runs
   inside AWS (EC2, Lambda, ECS) — there's then no secret that can leak.

## Multi-Factor Authentication (MFA)

MFA requires a second factor (a code from an authenticator app, or a
hardware key) in addition to a password. Enable it on the root user
immediately, and on every human IAM user:

```bash
# Create a virtual MFA device (do this once per user, via console is easiest
# since it needs to display/scan a QR code — CLI shown for completeness)
aws iam create-virtual-mfa-device \
  --virtual-mfa-device-name yourname-admin-mfa \
  --outfile qr-code.png \
  --bootstrap-method QRCodePNG

# After scanning the QR code in an authenticator app, enable it with two
# consecutive codes to prove the device is synced
aws iam enable-mfa-device \
  --user-name yourname-admin \
  --serial-number arn:aws:iam::123456789012:mfa/yourname-admin-mfa \
  --authentication-code1 123456 \
  --authentication-code2 654321
```

!!! warning "Root user MFA is non-negotiable"
    Enable MFA on the root user before doing anything else in a real
    account. It has no permission boundary, so a leaked root password
    without MFA is a complete account takeover.

## Cheat sheet

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

## Exercise

1. Write a policy JSON file that allows only `s3:GetObject` and
   `s3:ListBucket` on a bucket named after you (e.g.
   `arn:aws:s3:::yourname-training-bucket` and its `/*` objects).
2. Create that policy with `aws iam create-policy`.
3. Create a group `TrainingReadOnly`, attach your new policy to it, and add
   your `readonly-test` user (from Module 1's exercise) to the group.
4. Run `aws iam list-attached-group-policies --group-name TrainingReadOnly`
   to confirm the attachment.
5. Enable MFA on your main admin user via the console (scan the QR code with
   an authenticator app like Google Authenticator or Authy) and confirm
   `aws iam list-mfa-devices --user-name yourname-admin` shows one device.
