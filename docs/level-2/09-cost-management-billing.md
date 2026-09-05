# 09 · Cost Management & Billing

Every warning box in this course so far has ended with some version of
"clean up so you don't get billed" — this module gives you the tools to
actually see and control that spend systematically, instead of just
remembering to run `terminate-instances`. It covers **Budgets** (alerting
before you overspend), **Cost Explorer** (understanding what you already
spent, and on what), and **tagging** (attributing spend to a project,
team, or environment).

## Core concepts

| Concept | What it is |
|---|---|
| **Budget** | A threshold (cost, usage, or RI/Savings Plan coverage) that triggers a notification when crossed or forecast to be crossed. |
| **Cost Explorer** | A queryable view of historical spend, filterable/grouped by service, tag, account, etc. |
| **Cost allocation tag** | A resource tag (e.g. `Project=training`) activated for use in billing reports and Cost Explorer grouping. |
| **Cost and Usage Report (CUR)** | The most granular, complete billing export — a CSV/Parquet dump to S3, one row per line item. |
| **Forecast** | Cost Explorer's projection of future spend based on historical trend. |

## Create a budget with an alert

```bash
cat > budget.json << 'EOF'
{
  "BudgetName": "training-monthly",
  "BudgetType": "COST",
  "TimeUnit": "MONTHLY",
  "BudgetLimit": { "Amount": "50", "Unit": "USD" }
}
EOF

cat > notifications.json << 'EOF'
[{
  "Notification": {
    "NotificationType": "ACTUAL",
    "ComparisonOperator": "GREATER_THAN",
    "Threshold": 80,
    "ThresholdType": "PERCENTAGE"
  },
  "Subscribers": [{ "SubscriptionType": "EMAIL", "Address": "you@example.com" }]
}]
EOF

aws budgets create-budget \
  --account-id 123456789012 \
  --budget file://budget.json \
  --notifications-with-subscribers file://notifications.json
```

This alerts by email once actual spend crosses 80% of the configured
limit for the month — a budget **only notifies**; unless you separately
wire it to an automated action (e.g. an SNS topic, module 4, triggering a
Lambda that stops resources), crossing 100% does not stop anything from
running or accruing further cost.

## Add a forecast-based alert too

A budget can carry more than one notification — add a second entry to the
same `notifications.json` array before creating the budget (or call
`aws budgets create-notification` afterward to add one to an existing
budget):

```json
{
  "Notification": {
    "NotificationType": "FORECASTED",
    "ComparisonOperator": "GREATER_THAN",
    "Threshold": 100,
    "ThresholdType": "PERCENTAGE"
  },
  "Subscribers": [{ "SubscriptionType": "EMAIL", "Address": "you@example.com" }]
}
```

A `FORECASTED` notification warns you *before* the month ends, based on
projected trend — often more useful than an `ACTUAL` alert that only
fires once you've already crossed the line.

## Query historical cost with Cost Explorer

```bash
aws ce get-cost-and-usage \
  --time-period Start=2026-07-01,End=2026-08-01 \
  --granularity MONTHLY \
  --metrics "UnblendedCost" \
  --group-by Type=DIMENSION,Key=SERVICE \
  --query "ResultsByTime[0].Groups[].[Keys[0],Metrics.UnblendedCost.Amount]" \
  --output table
# ------------------------------------------------
# |  Amazon Elastic Compute Cloud  |  4.32        |
# |  Amazon Simple Storage Service |  0.09        |
# |  AWS Lambda                    |  0.00        |
```

```bash
aws ce get-cost-forecast \
  --time-period Start=2026-08-01,End=2026-09-01 \
  --metric UNBLENDED_COST \
  --granularity MONTHLY
```

Cost Explorer must be enabled once per payer account (a one-time console
toggle before its API works) and historical data can take up to 24 hours
to first populate after enabling. Note that `ce get-*` API calls
themselves carry a small per-request charge beyond a modest free monthly
allowance — the console UI is free to click around in; scripting frequent
automated `ce` API polling is where that cost shows up.

## Tag resources and activate cost allocation tags

```bash
aws resourcegroupstaggingapi tag-resources \
  --resource-arn-list \
      arn:aws:ec2:us-east-1:123456789012:instance/i-0123456789abcdef0 \
      arn:aws:s3:::training-cf-origin-2026 \
  --tags Project=training,Environment=dev
```

Tagging the resource is only half the job — cost allocation tags must
also be **activated** before Cost Explorer/CUR will group or filter by
them, and activation is console-only (Billing → Cost Allocation Tags),
not currently available as a plain `aws` CLI call. Once activated, newly
activated tags apply going forward, not retroactively to cost incurred
before activation.

```bash
# Group cost by an activated tag, once it's active
aws ce get-cost-and-usage \
  --time-period Start=2026-07-01,End=2026-08-01 \
  --granularity MONTHLY --metrics "UnblendedCost" \
  --group-by Type=TAG,Key=Project
```

## Setting up a full Cost and Usage Report

```bash
aws cur put-report-definition --report-definition '{
  "ReportName": "training-cur",
  "TimeUnit": "DAILY",
  "Format": "textORcsv",
  "Compression": "GZIP",
  "AdditionalSchemaElements": ["RESOURCES"],
  "S3Bucket": "training-cur-reports-2026",
  "S3Prefix": "cur",
  "S3Region": "us-east-1",
  "ReportVersioning": "OVERWRITE_REPORT"
}'
```

A CUR is the most granular cost data AWS produces — one row per resource
per hour/day, every discount and tag applied — and is the standard input
for third-party cost-analysis tools (or your own Athena queries over S3).
For a training account, Cost Explorer's grouping/filtering UI and API are
usually enough; reach for a CUR when you need line-item-level detail or
external tooling integration.

!!! warning "A budget alert is not a spending cap"
    Nothing in this module stops a resource from running once a budget
    threshold is crossed — Budgets is purely observational/notification
    unless you build automation on top of it (e.g. an SNS-triggered Lambda
    that stops or terminates tagged resources). For a training account,
    the real safety net is still disciplined manual cleanup after each
    module's exercises, exactly as every earlier module's teardown steps
    describe.

## Cheat sheet

| Command | Purpose |
|---|---|
| `aws budgets create-budget --notifications-with-subscribers file://F` | Create a cost budget with an email alert threshold. |
| `aws ce get-cost-and-usage --group-by Type=DIMENSION,Key=SERVICE` | See historical spend broken down by service. |
| `aws ce get-cost-and-usage --group-by Type=TAG,Key=NAME` | See spend broken down by an activated cost allocation tag. |
| `aws ce get-cost-forecast` | Project future spend from historical trend. |
| `aws resourcegroupstaggingapi tag-resources` | Apply tags across resources for later cost attribution. |
| `aws cur put-report-definition` | Set up a granular daily/hourly Cost and Usage Report to S3. |

## How It Actually Works

AWS's billing pipeline is itself a distributed, asynchronous system, which
is why cost data is never "live." Every billable action across every
service emits **usage records** into an internal metering pipeline; these
records are batched, aggregated by resource/usage-type/region, rated
against your account's pricing (list price, Reserved Instance/Savings Plan
discounts, volume tiers), and rolled up into the Cost Explorer/Cost and
Usage Report datasets on a delay that's typically several hours, sometimes
up to a day for less common usage types — this is why a just-terminated
resource can still show incomplete or estimated charges for a while, and why
Cost Explorer explicitly labels the current day's data as an estimate.

**Reserved Instances and Savings Plans** don't change which physical
resource you get — they're purely a billing-time optimization. AWS's
billing engine, when rating your usage records at the end of the process,
looks for eligible usage matching your commitment's attributes (instance
family/region for RIs, or a dollar-per-hour commitment for Savings Plans)
and *retroactively* applies the discounted rate to matching hourly usage —
this is why RIs/Savings Plans have no effect on performance or availability,
and why partial-month purchases still get pro-rated benefit for the hours
that remain: the discount is applied at settlement time, not reserved
capacity in the traditional sense (Regional RIs, notably, don't even
guarantee capacity — only the billing discount).

**Budgets and cost anomaly detection** poll this same lagging billing data
rather than intercepting API calls in real time, which is the underlying
reason a Budget alert can never prevent an overspend before it happens — it
can only notify you after usage has already been metered and aggregated,
making it a detection control, not a real-time enforcement control (Service
Quotas and IAM deny policies are the actual preventive mechanisms).

## Exercise

1. Create a monthly cost budget with an `ACTUAL` alert at 80% and a
   `FORECASTED` alert at 100%, both emailing you.
2. Tag at least 2 resources from earlier modules with `Project=training`,
   then (console step) activate `Project` as a cost allocation tag.
3. Once cost data exists, run `get-cost-and-usage` grouped by `SERVICE`
   for the current month and identify your top 2 cost drivers.
4. Run `get-cost-forecast` for next month and compare it to your budget
   limit.
5. After tag activation propagates, re-run `get-cost-and-usage` grouped by
   `Type=TAG,Key=Project` and confirm tagged resources' cost is broken out
   separately from untagged spend.
6. Delete the budget when you're done experimenting (budgets themselves
   have no cost, but keeping stale ones around clutters the account).
