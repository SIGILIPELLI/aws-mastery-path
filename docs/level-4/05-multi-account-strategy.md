# Multi-Account Strategy (Organizations, Control Tower)

Level 3 module 4 covered SCPs assuming an Organization already exists.
This module covers building that structure: why production workloads
end up spread across many AWS accounts rather than one, and how
**Control Tower** automates setting that up safely.

## Why multiple accounts

A single account sharing prod and dev workloads means one team's
mistake (deleting the wrong resource, hitting a service limit) can
affect another team's production traffic, and IAM policies to properly
separate them within one account get unmanageably complex. Separate
accounts give hard blast-radius boundaries for free — billing, quotas,
and IAM are already account-scoped.

## Organizations structure

```bash
aws organizations create-organization --feature-set ALL

aws organizations create-organizational-unit \
  --parent-id r-abcd \
  --name Workloads

aws organizations create-account \
  --account-name "training-app-prod" \
  --email aws-prod@example.com
# {
#     "CreateAccountStatus": { "Id": "car-abc123", "State": "IN_PROGRESS" }
# }

aws organizations describe-create-account-status --create-account-request-id car-abc123
# State: "SUCCEEDED", AccountId: "333333333333"

aws organizations move-account \
  --account-id 333333333333 \
  --source-parent-id r-abcd \
  --destination-parent-id ou-abcd-workloads1
```

A typical OU layout: `Security` (log archive, audit accounts),
`Infrastructure` (shared networking), `Workloads/Prod`,
`Workloads/NonProd`, `Sandbox` (personal experimentation, tightly
capped) — each OU carries its own SCPs.

## Control Tower: guardrails on top of Organizations

Control Tower automates the "landing zone" setup: a dedicated log
archive account, an audit account, baseline SCPs, and self-service
account provisioning through an **Account Factory**, so new accounts
are compliant from creation rather than configured by hand afterward.

```bash
aws controltower list-landing-zones

aws controltower get-landing-zone --landing-zone-identifier <arn>

# Enrolling an existing account brings it under Control Tower governance
aws controltower create-account-factory-account \
  --account-name "training-app-staging" \
  --account-email aws-staging@example.com \
  --organizational-unit-id ou-abcd-workloads1
```

Control Tower enforces **detective guardrails** (Config rules that flag
non-compliant resources) and **preventive guardrails** (SCPs that block
actions outright) — both defined once in the landing zone and applied
consistently as new accounts are enrolled, so you don't hand-write SCPs
per account.

## Cross-account resource sharing

Rather than duplicating a shared resource (like a Transit Gateway or an
ACM certificate) in every account, **Resource Access Manager (RAM)**
shares it directly:

```bash
aws ram create-resource-share \
  --name shared-tgw \
  --resource-arns arn:aws:ec2:us-east-1:111111111111:transit-gateway/tgw-0abc123 \
  --principals ou-abcd-workloads1
```

Accounts in `ou-abcd-workloads1` can now attach their VPCs to the
shared Transit Gateway without it being duplicated per account —
combining Level 3 module 1's TGW concept with the multi-account
boundary here.

## Centralized logging and billing

```bash
# CloudTrail organization trail — logs every account, created once
aws cloudtrail create-trail \
  --name org-trail \
  --s3-bucket-name org-cloudtrail-logs \
  --is-organization-trail \
  --is-multi-region-trail
```

An organization trail, created from the management account, applies to
every current and future member account automatically — individual
accounts cannot disable or modify it, which is exactly the point for
audit integrity.

## Gotchas

- **The management (root) account should run no workloads** — it's the
  billing and organization-control root; best practice is to keep it
  empty except for Organizations/Control Tower administration, since
  SCPs don't apply to it (as covered in Level 3 module 4).
- **Account creation is asynchronous and can fail silently on quota
  limits** — always poll `describe-create-account-status`; a failed
  creation doesn't throw from the initial `create-account` call.
- **Closing an account has a mandatory ~90-day cooling-off period**
  before you can reuse its email address for a new account — plan
  account naming/emails with this in mind.
- **RAM-shared resources are still owned and billed to the sharing
  account** — the consuming account gets usage, not ownership; deleting
  the resource share revokes access immediately for all consumers.
- **Control Tower's Account Factory accounts come with default network
  configuration (VPC/subnets)** that may not match your actual
  requirements — treat it as a starting point, not a final network
  design.

## Cheat sheet

| Command | Purpose |
|---|---|
| `aws organizations create-organization` | Bootstrap an Organization |
| `aws organizations create-account` | Provision a new member account |
| `aws organizations move-account` | Reorganize OU structure |
| `aws controltower create-account-factory-account` | Provision a governed account |
| `aws ram create-resource-share` | Share a resource across accounts |
| `aws cloudtrail create-trail --is-organization-trail` | Org-wide audit logging |

## Exercise

Design an OU structure (on paper) for an organization with three teams
each needing prod/non-prod separation, plus one shared networking
account. List which SCPs (from Level 3 module 4) you'd attach at the
`Workloads` OU vs. only at `Workloads/Prod`, and which resource you'd
share via RAM from the networking account.

## Note on execution

This module was written and verified via syntax review only — no
Organization, Control Tower landing zone, or accounts were actually
created, since doing so has real billing and irreversible account-
lifecycle implications.
