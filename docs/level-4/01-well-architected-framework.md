# AWS Well-Architected Framework

Every module in this path has touched pieces of good architecture —
IAM scoping, monitoring, DR. The **Well-Architected Framework** gives
you a structured way to evaluate an existing system against six
pillars, using a standard set of questions rather than ad-hoc review.

## The six pillars

| Pillar | Core question |
|---|---|
| Operational Excellence | Can you run and monitor systems, and continuously improve? |
| Security | Is data and infrastructure protected, with least privilege? |
| Reliability | Does the system recover from failure and scale to meet demand? |
| Performance Efficiency | Are resources used efficiently, and can they adapt as needs change? |
| Cost Optimization | Are you spending money only where it delivers value? |
| Sustainability | Are you minimizing environmental impact per unit of work? |

Trade-offs are explicit and expected — e.g., higher reliability
(multi-AZ, multi-region) usually costs more; a Well-Architected review
makes you state the trade-off deliberately rather than by accident.

## Running a review with the CLI

The Well-Architected Tool lets you record a workload and answer its
standard question set programmatically.

```bash
aws wellarchitected create-workload \
  --workload-name "training-app-prod" \
  --description "Customer-facing order processing API" \
  --environment PRODUCTION \
  --aws-regions us-east-1 \
  --lenses wellarchitected \
  --review-owner platform-team@example.com
# { "WorkloadId": "abcd1234ef567890abcd1234ef567890", "WorkloadArn": "arn:aws:wellarchitected:..." }

aws wellarchitected list-answers \
  --workload-id abcd1234ef567890abcd1234ef567890 \
  --lens-alias wellarchitected \
  --pillar-id security
```

Each pillar's questions come with **choices** you mark as applicable;
unanswered or "not applicable without justification" choices surface as
**risks**:

```bash
aws wellarchitected update-answer \
  --workload-id abcd1234ef567890abcd1234ef567890 \
  --lens-alias wellarchitected \
  --question-id sec_iam_least_privilege \
  --selected-choices sec_permissions_boundaries,sec_scp
```

```bash
aws wellarchitected get-lens-review-report \
  --workload-id abcd1234ef567890abcd1234ef567890 \
  --lens-alias wellarchitected
# Returns a base64-encoded PDF summarizing high/medium risks per pillar
```

## High-risk issues (HRIs)

The tool flags each unaddressed question as a **High Risk Issue (HRI)**
or **Medium Risk Issue (MRI)** based on the choices selected. A review's
output is a prioritized list, not a pass/fail score — the goal is
tracking risk down over time, revisited periodically (AWS recommends
at least every 6-12 months or after major architecture changes).

```bash
aws wellarchitected list-lens-review-improvements \
  --workload-id abcd1234ef567890abcd1234ef567890 \
  --lens-alias wellarchitected \
  --pillar-id reliability
```

## Applying it: a worked example

A workload with a single-AZ RDS instance and no automated backups
would answer "no" to reliability questions about withstanding
component failure — an HRI. The remediation ties directly to Level 1
(automated backups) and Level 3 module 7 (multi-AZ, DR strategy):

```bash
aws rds modify-db-instance \
  --db-instance-identifier prod-db \
  --multi-az \
  --backup-retention-period 7 \
  --apply-immediately
```

## Gotchas

- **The Framework is a review methodology, not an enforcement
  mechanism** — nothing stops you from deploying architecture that
  fails every pillar; it only surfaces risk when someone runs a
  review. Pair it with automated guardrails (SCPs, Config rules) for
  actual enforcement.
- **"Not applicable" needs written justification** — the tool lets you
  mark a question N/A, but skipping this step for convenience defeats
  the purpose; reviewers should challenge N/A answers that lack
  reasoning.
- **Lenses beyond the base Well-Architected lens exist** (Serverless,
  SaaS, Machine Learning lenses) and ask domain-specific questions —
  applying only the generic lens to a serverless workload misses
  relevant questions.
- **Reviews go stale** — a workload marked "reviewed" six months ago
  after a major refactor no longer reflects reality; treat the
  `UpdatedAt` timestamp on a workload as an expiration warning, not a
  badge.
- **Milestones matter for tracking improvement** — `create-milestone`
  snapshots the current risk state; without milestones you can't show
  whether risk actually decreased between reviews.

## Cheat sheet

| Command | Purpose |
|---|---|
| `aws wellarchitected create-workload` | Register a workload for review |
| `aws wellarchitected update-answer` | Record answers to pillar questions |
| `aws wellarchitected list-lens-review-improvements` | List HRIs/MRIs with remediation guidance |
| `aws wellarchitected create-milestone` | Snapshot current risk state |
| `aws wellarchitected get-lens-review-report` | Generate a PDF summary |

## How It Actually Works

The Well-Architected Tool isn't a static checklist grader — each pillar's
questions map to a structured evaluation of your architecture against risk
patterns AWS has observed at scale across its own customer base, and the
"High Risk Issue" flags it produces are derived from real, mechanism-level
failure modes described elsewhere in this course: a Reliability pillar flag
for "single points of failure," for instance, is checking for exactly the
kind of thing you saw in the RDS Multi-AZ or ALB health-check modules —
whether a component's failure mode has an automated, tested path to
recovery, not just a redundant copy sitting idle and unverified.

The **Reliability** pillar's emphasis on "test recovery procedures
regularly" exists because most of the automated resilience mechanisms this
course covers (RDS Multi-AZ failover, ASG health-check replacement, Route
53 failover routing) have failure detection thresholds and cutover
mechanics that only get exercised during a real failure — untested failover
paths are a well-documented cause of DR mechanisms silently failing when
actually needed (a misconfigured health check grace period, a stale
security group blocking the standby), which is why "chaos engineering"
(covered later in this level) exists specifically to exercise these paths
deliberately rather than waiting for a real outage to find the gap.

The **Cost Optimization** pillar's "right-sizing" guidance connects directly
to how billing is metered (module 09, Level 2): because AWS bills per
resource-hour of *provisioned* capacity regardless of utilization,
right-sizing isn't a vague efficiency suggestion — it's a mechanical
consequence of the billing model rewarding you for matching provisioned
capacity to the CloudWatch-observed utilization curve rather than
over-provisioning against worst-case guesses.

## Exercise

Pick any architecture you've built in this path (e.g., the Level 3
CI/CD project) and manually walk through five Security-pillar
questions from the Well-Architected Tool (create a workload via CLI to
see the real question set). For each, note whether you'd mark it
resolved, an HRI, or N/A, and what concrete change would resolve any
HRIs.
