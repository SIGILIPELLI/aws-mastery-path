# Advanced IAM (SCPs, Permission Boundaries)

Level 1 covered IAM users, roles, and policies within a single account.
Once an organization runs multiple AWS accounts, two more IAM
mechanisms matter: **Service Control Policies (SCPs)**, which cap what
an entire account (or OU) can ever do, and **permission boundaries**,
which cap what an individual role or user can be granted — even by an
administrator.

## SCPs: guardrails at the account/OU level

SCPs live in **AWS Organizations** and apply to member accounts. They
never *grant* permissions — they only set a ceiling. A user with
`AdministratorAccess` in an account still can't perform an action an
SCP denies.

```bash
aws organizations create-policy \
  --name DenyRegionLockdown \
  --type SERVICE_CONTROL_POLICY \
  --content file://deny-non-us-regions.json
```

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyOutsideAllowedRegions",
      "Effect": "Deny",
      "NotAction": ["iam:*", "organizations:*", "route53:*", "support:*"],
      "Resource": "*",
      "Condition": {
        "StringNotEquals": { "aws:RequestedRegion": ["us-east-1", "us-west-2"] }
      }
    }
  ]
}
```

```bash
aws organizations attach-policy \
  --policy-id p-abc12345 \
  --target-id ou-root-1a2b3c4d
```

`NotAction` combined with `Deny` is a common SCP pattern: deny
everything **except** a short list of global/free services, in every
region except the approved ones.

## Permission boundaries

A boundary is a managed policy attached to a *role or user* (not an
account) that caps its **maximum possible permissions**, regardless of
what identity-based policies are later attached. It's how you let a
team create their own IAM roles without risking privilege escalation.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:*", "lambda:*", "logs:*"],
      "Resource": "*"
    }
  ]
}
```

```bash
aws iam create-role \
  --role-name dev-created-role \
  --assume-role-policy-document file://trust-policy.json \
  --permissions-boundary arn:aws:iam::123456789012:policy/DeveloperBoundary
```

Even if someone later attaches `AdministratorAccess` to
`dev-created-role`, its **effective** permissions are the intersection
of the attached policy and the boundary — so it's still capped at
S3/Lambda/Logs.

## Effective permissions: the intersection rule

An action is allowed only if **all** of these agree:
1. No SCP in the account's OU chain denies it.
2. The identity's permission boundary (if any) allows it.
3. An identity-based or resource-based policy explicitly allows it.
4. No identity-based policy explicitly denies it.

An explicit `Deny` anywhere in this chain always wins.

## Cross-account role assumption

```bash
# In the target account: trust policy on the role being assumed
```
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "AWS": "arn:aws:iam::111111111111:root" },
    "Action": "sts:AssumeRole",
    "Condition": { "StringEquals": { "sts:ExternalId": "shared-secret-123" } }
  }]
}
```

```bash
aws sts assume-role \
  --role-arn arn:aws:iam::222222222222:role/CrossAccountReadOnly \
  --role-session-name audit-session \
  --external-id shared-secret-123
# Returns temporary AccessKeyId/SecretAccessKey/SessionToken (default 1hr)
```

`ExternalId` prevents the "confused deputy" problem — without it, a
third party who's given the same role ARN by two different customers
could accidentally trigger cross-customer access.

## Gotchas

- **SCPs don't apply to the management (root) account of an
  Organization** — a common mistake is testing lockdown SCPs from the
  management account and concluding they don't work.
- **`FullAWSAccess` is the default SCP** attached to every OU; removing
  it without an explicit allow-list SCP in place locks out every
  action, including the console.
- **Permission boundaries silently cap, they don't error loudly** — a
  role that seems to have `AdministratorAccess` but still gets
  `AccessDenied` is almost always hitting its boundary, not a missing
  grant. Check `aws iam get-role --query PermissionsBoundary`.
- **Assumed-role temporary credentials expire** (default 1 hour, up to
  12 with `--duration-seconds`) — long-running jobs need to refresh,
  not just fetch once at startup.
- **SCP evaluation is per-OU-chain**, so a policy attached at a parent
  OU affects every nested OU and account below it — test in a sandbox
  OU before attaching org-wide.

## Cheat sheet

| Mechanism | Applies to | Can grant permissions? |
|---|---|---|
| Identity-based policy | User/role | Yes |
| Resource-based policy | Resource (e.g. S3 bucket) | Yes |
| Permission boundary | User/role | No — caps only |
| SCP | Account/OU | No — caps only |
| `sts:AssumeRole` + `ExternalId` | Cross-account | N/A (auth mechanism) |

## Exercise

Write an SCP that denies `ec2:RunInstances` for any instance type other
than `t3.micro` and `t3.small` account-wide (hint: use a `Condition` on
`ec2:InstanceType`), and a permission boundary that would let a
developer role manage its own Lambda functions but nothing else. Note
which of the two would stop someone from launching an `m5.24xlarge`.
