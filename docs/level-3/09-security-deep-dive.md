# Security Deep Dive (GuardDuty, WAF, Security Hub)

Level 1's security group and IAM basics stop intrusions at the network
and permission edge. This module covers three services that detect and
respond to threats that get past those edges: **GuardDuty** (threat
detection), **WAF** (application-layer filtering), and **Security Hub**
(centralized findings).

## GuardDuty: continuous threat detection

GuardDuty analyzes VPC Flow Logs, DNS logs, and CloudTrail events
against threat intelligence feeds and anomaly models — no agents to
install, no logs to configure yourself.

```bash
aws guardduty create-detector --enable

aws guardduty list-findings \
  --detector-id 8ab1c2d3e4f5678901234567890abcdef \
  --finding-criteria '{"Criterion":{"severity":{"Gte":7}}}'

aws guardduty get-findings \
  --detector-id 8ab1c2d3e4f5678901234567890abcdef \
  --finding-ids 8cb1c2d3e4f5678901234567890abcdef
# {
#     "Findings": [{
#         "Type": "UnauthorizedAccess:EC2/SSHBruteForce",
#         "Severity": 5.0,
#         "Title": "203.0.113.5 is performing SSH brute force attacks against i-0abc123."
#     }]
# }
```

Findings are scored 1-10 (severity); route high-severity findings to
EventBridge for automated response:

```bash
aws events put-rule \
  --name guardduty-high-severity \
  --event-pattern '{"source":["aws.guardduty"],"detail":{"severity":[{"numeric":[">=",7]}]}}'
```

## WAF: filtering at the application layer

Security groups filter by IP/port; **WAF** inspects HTTP request
content — SQL injection patterns, rate limits per IP, geographic
blocks — and attaches to CloudFront, ALB, or API Gateway.

```bash
aws wafv2 create-web-acl \
  --name training-app-waf \
  --scope REGIONAL \
  --default-action Allow={} \
  --visibility-config SampledRequestsEnabled=true,CloudWatchMetricsEnabled=true,MetricName=training-app-waf \
  --rules file://waf-rules.json
```

```json
[
  {
    "Name": "RateLimitPerIP",
    "Priority": 1,
    "Action": { "Block": {} },
    "Statement": { "RateBasedStatement": { "Limit": 2000, "AggregateKeyType": "IP" } },
    "VisibilityConfig": { "SampledRequestsEnabled": true, "CloudWatchMetricsEnabled": true, "MetricName": "RateLimitPerIP" }
  },
  {
    "Name": "AWSManagedCommonRuleSet",
    "Priority": 2,
    "OverrideAction": { "None": {} },
    "Statement": { "ManagedRuleGroupStatement": { "VendorName": "AWS", "Name": "AWSManagedRulesCommonRuleSet" } },
    "VisibilityConfig": { "SampledRequestsEnabled": true, "CloudWatchMetricsEnabled": true, "MetricName": "CommonRuleSet" }
  }
]
```

```bash
aws wafv2 associate-web-acl \
  --web-acl-arn arn:aws:wafv2:us-east-1:123456789012:regional/webacl/training-app-waf/abc123 \
  --resource-arn arn:aws:elasticloadbalancing:us-east-1:123456789012:loadbalancer/app/training-alb/def456
```

AWS Managed Rule Groups (like `AWSManagedRulesCommonRuleSet`, covering
OWASP Top 10 patterns) save you from writing your own SQLi/XSS regex
rules — start with `OverrideAction: None` (enforce) only after testing
with `Count` to see what would have been blocked.

## Security Hub: centralized findings

GuardDuty, Inspector, Macie, and WAF each produce their own findings.
Security Hub aggregates them (plus your own custom findings) into one
place, scored against standards like CIS AWS Foundations.

```bash
aws securityhub enable-security-hub \
  --enabled-standards StandardsSubscriptionArns=arn:aws:securityhub:us-east-1::standards/aws-foundational-security-best-practices/v/1.0.0

aws securityhub get-findings \
  --filters '{"SeverityLabel":[{"Value":"CRITICAL","Comparison":"EQUALS"}]}'
```

In a multi-account Organization, one account can be designated the
**Security Hub administrator**, aggregating findings from every member
account into a single dashboard — set up via
`aws securityhub create-members` plus an invitation/auto-enable flow.

## Gotchas

- **GuardDuty has no free tier beyond a 30-day trial** and bills on
  volume of logs analyzed — a very high-traffic VPC can make GuardDuty
  itself a meaningful cost line; it's still almost always worth it, but
  don't enable it silently across every account without checking the
  estimate first (`get-usage-statistics`).
- **WAF's default action matters more than the rules** — a Web ACL with
  `DefaultAction: Allow` and no matching rule lets everything through;
  double check `default-action` isn't accidentally `Block` in
  production either, which would 500 all legitimate traffic.
- **Rule priority order determines evaluation** — WAF evaluates rules
  in ascending `Priority` and stops at the first terminating action
  (`Block`/`Allow`), so a broad allow rule placed before a specific
  block rule silently defeats it.
- **Security Hub standards checks run periodically, not real-time** —
  don't expect a finding to appear the instant a misconfiguration
  happens; checks run on an internal schedule (typically within hours).
- **Enabling multiple standards multiplies findings volume** — start
  with one (e.g., AWS Foundational Security Best Practices) and
  triage before turning on CIS and PCI DSS simultaneously.

## Cheat sheet

| Command | Purpose |
|---|---|
| `aws guardduty create-detector --enable` | Turn on threat detection |
| `aws guardduty list-findings` | List current findings |
| `aws wafv2 create-web-acl` | Create a Web ACL |
| `aws wafv2 associate-web-acl` | Attach it to ALB/CloudFront/API Gateway |
| `aws securityhub enable-security-hub` | Turn on centralized findings |
| `aws securityhub get-findings` | Query aggregated findings |

## Exercise

Enable GuardDuty in a test account, create a WAF Web ACL with the AWS
Managed Common Rule Set in `Count` mode (not enforcing) attached to an
ALB, and enable Security Hub with the AWS Foundational Security Best
Practices standard. After an hour, compare what each of the three
surfaces reports for the same account.
