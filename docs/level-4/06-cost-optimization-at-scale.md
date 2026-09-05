# Cost Optimization at Scale

Small workloads can eyeball their AWS bill. At scale — hundreds of
instances, many accounts — cost optimization becomes systematic:
committing to predictable usage for a discount, using interruptible
capacity where possible, and continuously rightsizing based on real
utilization data.

## Savings Plans vs. Reserved Instances vs. Spot

| | Flexibility | Discount vs. on-demand | Commitment |
|---|---|---|---|
| On-Demand | Full | None (baseline) | None |
| Savings Plans | Compute or EC2-instance-family scoped | Substantial | 1 or 3 years, $/hour commitment |
| Reserved Instances | Locked to instance family/region (Standard) or flexible (Convertible) | Substantial, similar to Savings Plans | 1 or 3 years |
| Spot Instances | Interruptible, any workload tolerant of termination | Largest | None — priced by market, can be reclaimed |

**Compute Savings Plans** are the most flexible commitment — they apply
across EC2, Fargate, and Lambda regardless of instance family or
region, in exchange for a slightly lower discount than instance-family-
locked EC2 Savings Plans or Standard RIs.

```bash
aws ce get-savings-plans-purchase-recommendation \
  --savings-plans-type COMPUTE_SP \
  --term-in-years ONE_YEAR \
  --payment-option NO_UPFRONT \
  --lookback-period-in-days SIXTY_DAYS

aws savingsplans create-savings-plan \
  --savings-plan-offering-id offering-abc123 \
  --commitment 10.00
```

Cost Explorer's `get-savings-plans-purchase-recommendation` analyzes
your last 30/60 days of usage and recommends a commitment level that
covers steady-state usage without over-committing to capacity you
might not use.

## Spot for interruption-tolerant workloads

```bash
aws ec2 request-spot-fleet --spot-fleet-request-config file://spot-fleet.json
```

```json
{
  "IamFleetRole": "arn:aws:iam::123456789012:role/aws-ec2-spot-fleet-role",
  "AllocationStrategy": "capacityOptimized",
  "TargetCapacity": 20,
  "LaunchSpecifications": [
    { "InstanceType": "m5.large", "SubnetId": "subnet-0aaa111" },
    { "InstanceType": "m5a.large", "SubnetId": "subnet-0aaa111" },
    { "InstanceType": "m6i.large", "SubnetId": "subnet-0aaa111" }
  ]
}
```

`capacityOptimized` allocation picks spot pools with the deepest
available capacity (lowest interruption risk) rather than the cheapest
price — for batch jobs and stateless workers, mixing several instance
types/families widens the pool AWS can draw from and reduces
simultaneous interruption risk.

## Rightsizing with Compute Optimizer

```bash
aws compute-optimizer get-ec2-instance-recommendations \
  --instance-arns arn:aws:ec2:us-east-1:123456789012:instance/i-0abc123def456

# {
#     "instanceRecommendations": [{
#         "currentInstanceType": "m5.2xlarge",
#         "finding": "OVER_PROVISIONED",
#         "recommendationOptions": [{ "instanceType": "m5.large", "performanceRisk": 1.2 }]
#     }]
# }
```

Compute Optimizer analyzes 14+ days of CloudWatch utilization and flags
instances as `OVER_PROVISIONED`, `UNDER_PROVISIONED`, or
`OPTIMIZED` — turning "does this need to be this big" from a guess into
a data-backed recommendation across an entire fleet at once.

## Cost allocation and anomaly detection

```bash
# Tag-based cost tracking — activate a cost allocation tag
aws ce update-cost-allocation-tags-status \
  --cost-allocation-tags-status TagKey=team,Status=Active

aws ce get-cost-and-usage \
  --time-period Start=2026-08-01,End=2026-08-25 \
  --granularity DAILY \
  --metrics UnblendedCost \
  --group-by Type=TAG,Key=team

aws ce create-anomaly-monitor \
  --anomaly-monitor '{"MonitorName":"prod-monitor","MonitorType":"DIMENSIONAL","MonitorDimension":"SERVICE"}'
```

Cost Anomaly Detection uses ML on historical spend patterns to flag
unusual spikes per service/account automatically, rather than someone
noticing a surprise on the monthly bill weeks later.

## S3 storage class tiering

```bash
aws s3api put-bucket-lifecycle-configuration \
  --bucket training-datalake-raw \
  --lifecycle-configuration '{
    "Rules": [{
      "ID": "tier-old-data",
      "Status": "Enabled",
      "Filter": { "Prefix": "orders/" },
      "Transitions": [
        { "Days": 30, "StorageClass": "STANDARD_IA" },
        { "Days": 90, "StorageClass": "GLACIER" }
      ]
    }]
  }'
```

## Gotchas

- **Savings Plans commit to a dollar-per-hour spend, not specific
  resources** — under-forecasting usage wastes the commitment (you pay
  it whether or not you use it); over-committing based on a temporary
  spike locks in cost for months or years.
- **Spot capacity can disappear entirely for a given instance type/AZ
  combination** — a fleet requesting only one instance type in one AZ
  has no fallback; diversify across types and AZs as shown above.
- **Compute Optimizer needs enrollment and 14+ days of data** — a
  freshly launched account or instance won't have recommendations yet;
  it isn't real-time.
- **Storage class transitions aren't free** — Glacier retrieval has its
  own cost and multi-hour to multi-day latency depending on retrieval
  tier; lifecycle rules on frequently-accessed data can backfire into
  higher total cost from repeated retrievals.
- **Tag-based cost allocation only tracks resources tagged *after* the
  tag is activated** — retroactive tagging doesn't retroactively
  populate Cost Explorer's historical breakdown.

## Cheat sheet

| Task | Command |
|---|---|
| Get Savings Plan recommendation | `aws ce get-savings-plans-purchase-recommendation` |
| Purchase a Savings Plan | `aws savingsplans create-savings-plan` |
| Request diversified spot capacity | `aws ec2 request-spot-fleet` |
| Get rightsizing recommendations | `aws compute-optimizer get-ec2-instance-recommendations` |
| Group costs by tag | `aws ce get-cost-and-usage --group-by Type=TAG` |
| Detect spend anomalies | `aws ce create-anomaly-monitor` |
| Tier S3 storage automatically | `aws s3api put-bucket-lifecycle-configuration` |

## How It Actually Works

At scale, cost optimization is fundamentally about exploiting how each
service's billing meter actually works, not just "using less." **Spot
Instances** are spare EC2 capacity AWS's own internal capacity-management
system has determined it doesn't currently need for On-Demand/Reserved
commitments — pricing floats based on real-time supply and demand per
instance pool (instance type + AZ), and the two-minute interruption notice
you get before reclamation exists because AWS's capacity system needs that
resource back for a higher-priority (On-Demand/Reserved) request, not
because of anything you did; this is why Spot strategy at scale means
diversifying across many instance-pool combinations (so losing one pool's
capacity doesn't take down your whole fleet) rather than picking the single
cheapest option.

**Savings Plans** work through the billing engine's rating step (module 09,
Level 2) matching your committed dollar-per-hour rate against *any*
eligible usage first, before falling back to on-demand rates for the
remainder — at fleet scale, this means Savings Plans should be sized against
your aggregate, sustained baseline usage across the whole organization
(often via a payer account applying discounts across linked accounts),
because the discount-matching happens organization-wide during the same
consolidated billing rating pass, not account-by-account.

**S3 Intelligent-Tiering** at scale works by S3 itself monitoring
per-object access patterns and automatically moving objects between access
tiers based on observed (not predicted) usage — this is a real, continuous
background process S3 runs against your bucket's actual request logs
internally, which is why it has no retrieval fees between the frequent and
infrequent tiers (unlike manually choosing Standard-IA) but does carry a
small per-object monitoring fee: you're paying for that continuous
tier-decision computation instead of doing the tiering analysis yourself.

## Exercise

Using `aws compute-optimizer get-ec2-instance-recommendations` (or the
console if CLI access isn't enrolled), find one instance flagged
`OVER_PROVISIONED` in a test account, and write out the exact
`aws ec2 modify-instance-attribute` command you'd run to resize it to
the recommended type during a maintenance window.
