# Compliance & Governance (Config, Audit Manager)

Level 3 module 9 covered detecting active threats. Compliance is a
related but distinct concern: continuously verifying that resources
*stay* configured the way policy requires (encryption enabled, no
public S3 buckets), and producing the evidence auditors need to prove
it — before an auditor or an attacker finds the gap.

## AWS Config: continuous configuration tracking

Config records every resource's configuration and every change to it,
and evaluates resources against **rules** you define or select from
AWS-managed rules.

```bash
aws configservice put-configuration-recorder \
  --configuration-recorder name=default,roleARN=arn:aws:iam::123456789012:role/ConfigRole \
  --recording-group allSupported=true,includeGlobalResourceTypes=true

aws configservice put-delivery-channel \
  --delivery-channel name=default,s3BucketName=training-config-bucket

aws configservice start-configuration-recorder --configuration-recorder-name default
```

```bash
aws configservice put-config-rule --config-rule '{
  "ConfigRuleName": "s3-bucket-public-read-prohibited",
  "Source": { "Owner": "AWS", "SourceIdentifier": "S3_BUCKET_PUBLIC_READ_PROHIBITED" }
}'

aws configservice get-compliance-details-by-config-rule \
  --config-rule-name s3-bucket-public-read-prohibited
# {
#     "EvaluationResults": [
#         { "EvaluationResultIdentifier": { "EvaluationResultQualifier": { "ResourceId": "training-app-bucket-8842" } }, "ComplianceType": "NON_COMPLIANT" }
#     ]
# }
```

Config rules can be evaluated **on change** (near-real-time) or on a
**periodic** schedule — a NON_COMPLIANT finding tells you exactly
which resource drifted and when, unlike a point-in-time manual audit.

## Auto-remediation

```bash
aws configservice put-remediation-configurations --remediation-configurations '[{
  "ConfigRuleName": "s3-bucket-public-read-prohibited",
  "TargetType": "SSM_DOCUMENT",
  "TargetId": "AWSConfigRemediation-RemovePublicAccessBlockFromBucket",
  "Automatic": true,
  "MaximumAutomaticAttempts": 3
}]'
```

Automatic remediation via an SSM Automation document closes the loop:
detect drift, then fix it without a human — appropriate for
well-understood, low-risk fixes (block public access) but risky for
anything that could disrupt a legitimate configuration; test in
`Automatic: false` mode first and review before enabling auto-apply.

## Conformance packs: rule sets as a unit

```bash
aws configservice put-conformance-pack \
  --conformance-pack-name operational-best-practices-for-pci-dss \
  --template-s3-uri s3://aws-configservice-conformancepack-templates/Operational-Best-Practices-for-PCI-DSS.yaml
```

AWS publishes ready-made conformance packs mapping to common
frameworks (PCI DSS, HIPAA, NIST) — bundling dozens of individual Config
rules that implement that framework's technical controls, deployable
as one unit per account or organization-wide.

## Audit Manager: evidence collection

Config tells you compliance state right now. **Audit Manager** builds
the audit trail over time — continuously collecting evidence (Config
snapshots, CloudTrail events, manual attestations) mapped to a
framework's specific controls, so an audit doesn't require weeks of
manually gathering screenshots.

```bash
aws auditmanager create-assessment \
  --name "PCI-DSS-2026-Q3" \
  --framework-id abc123-pci-dss-framework \
  --scope '{"awsAccounts":[{"id":"123456789012"}]}' \
  --roles '{"roleType":"PROCESS_OWNER","roleArn":"arn:aws:iam::123456789012:role/AuditOwner"}'

aws auditmanager get-evidence-by-evidence-folder \
  --assessment-id assess-abc123 \
  --control-set-id cs-encryption \
  --evidence-folder-id ef-def456
```

Each control in the framework accumulates evidence automatically as
resources are created/changed — at audit time, you export a report
rather than reconstructing history from logs after the fact.

## Gotchas

- **Config recording has a per-resource-configuration-item cost** — a
  large, frequently-changing account (e.g., an Auto Scaling group
  churning instances) can generate significant Config evidence volume
  and cost; scope recording to resource types you actually need
  evaluated if cost is a concern.
- **Auto-remediation can fight legitimate configuration** — a
  temporarily public bucket for a legitimate one-time file share gets
  auto-remediated back to private the moment Config evaluates it,
  possibly disrupting an intentional, time-boxed exception.
- **Conformance packs are a starting point, not certification** —
  passing every rule in a PCI DSS conformance pack does not mean you
  are PCI DSS certified; certification requires a Qualified Security
  Assessor's independent review, which Audit Manager evidence
  supports but doesn't replace.
- **Config rules evaluate resource configuration, not runtime
  behavior** — a bucket policy that's technically "compliant" but
  overly permissive in an unanticipated way (e.g., a wildcard
  principal) may not trip a rule that only checks for the specific
  `PublicAccessBlock` setting.
- **Multi-account aggregation requires an explicit aggregator
  resource** (`aws configservice put-configuration-aggregator`) — Config
  is account/region-scoped by default; an org-wide compliance view
  needs this set up deliberately, similar to Security Hub's
  administrator account pattern from Level 3 module 9.

## Cheat sheet

| Command | Purpose |
|---|---|
| `aws configservice put-configuration-recorder` | Start tracking resource configs |
| `aws configservice put-config-rule` | Add a compliance rule |
| `aws configservice get-compliance-details-by-config-rule` | Check compliance status |
| `aws configservice put-remediation-configurations` | Auto-fix non-compliant resources |
| `aws configservice put-conformance-pack` | Deploy a framework's rule bundle |
| `aws auditmanager create-assessment` | Start continuous evidence collection |

## Exercise

Enable AWS Config in a test account, add the
`s3-bucket-server-side-encryption-enabled` managed rule, and create an
S3 bucket without default encryption to confirm it's flagged
NON_COMPLIANT. Then add a remediation configuration (in manual, not
automatic, mode) and trigger it by hand to bring the bucket into
compliance.
