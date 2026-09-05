# 08 · CloudWatch Monitoring & Logging

CloudWatch is AWS's monitoring and observability service: every service
you've touched so far — EC2, RDS, Lambda — already emits metrics to it
automatically, and Lambda's `print()`/`logging` output already lands in
CloudWatch Logs without any setup on your part. This module covers reading
those metrics and logs, setting an alarm that notifies you when something
crosses a threshold, and structuring log groups so they're useful once you
have more than one function or instance running.

## Metrics: numbers over time

Every AWS service publishes **metrics** — numeric data points over time,
grouped into **namespaces** (e.g. `AWS/Lambda`, `AWS/EC2`, `AWS/RDS`).

```bash
# List what metrics exist for your Lambda function
aws cloudwatch list-metrics \
  --namespace AWS/Lambda \
  --dimensions Name=FunctionName,Value=training-hello

# Pull actual data points: invocation count over the last hour, in 5-min buckets
aws cloudwatch get-metric-statistics \
  --namespace AWS/Lambda \
  --metric-name Invocations \
  --dimensions Name=FunctionName,Value=training-hello \
  --start-time "$(date -u -v-1H +%Y-%m-%dT%H:%M:%S)" \
  --end-time "$(date -u +%Y-%m-%dT%H:%M:%S)" \
  --period 300 \
  --statistics Sum
```

Common metrics you'll use constantly:

| Namespace | Metric | Meaning |
|---|---|---|
| `AWS/EC2` | `CPUUtilization` | Percent CPU used by an instance. |
| `AWS/Lambda` | `Invocations`, `Errors`, `Duration`, `Throttles` | Call count, failures, execution time, concurrency limits hit. |
| `AWS/RDS` | `CPUUtilization`, `FreeStorageSpace`, `DatabaseConnections` | Database health. |
| `AWS/ApplicationELB` | `RequestCount`, `HTTPCode_Target_5XX_Count` | Load balancer traffic and error rate. |

## Log groups and log streams

Every Lambda invocation's `print()`/`logging` output, and anything you
explicitly ship from EC2 via the CloudWatch agent, lands in a **log
group** — Lambda creates one automatically per function, named
`/aws/lambda/<function-name>`.

```bash
aws logs describe-log-groups --log-group-name-prefix /aws/lambda/training-hello

# List the individual invocation streams within that group
aws logs describe-log-streams \
  --log-group-name /aws/lambda/training-hello \
  --order-by LastEventTime --descending --max-items 5

# Tail the most recent log events (great for live-debugging a Lambda function)
aws logs tail /aws/lambda/training-hello --follow
```

Add a log line to the function from Module 7 and watch it appear live:

```python
import json, logging
logger = logging.getLogger()
logger.setLevel(logging.INFO)

def handler(event, context):
    logger.info(f"Received event: {json.dumps(event)}")
    ...
```

## Log retention

By default, Lambda's auto-created log groups **never expire** — logs
accumulate (and cost storage) forever unless you set a retention policy:

```bash
aws logs put-retention-policy \
  --log-group-name /aws/lambda/training-hello \
  --retention-in-days 14
```

Setting a sane retention period (7–30 days is typical for dev/training
work) on every log group you create is a cheap habit that avoids a slow
storage-cost creep.

## Alarms: get notified automatically

A CloudWatch **alarm** watches a metric and changes state (`OK` →
`ALARM`) when it crosses a threshold, optionally triggering a notification
via SNS (covered fully in Level 2).

```bash
# First, an SNS topic + your email subscribed to it
aws sns create-topic --name training-alerts
# TopicArn: arn:aws:sns:us-east-1:123456789012:training-alerts

aws sns subscribe \
  --topic-arn arn:aws:sns:us-east-1:123456789012:training-alerts \
  --protocol email \
  --notification-endpoint you@example.com
# Check your inbox and confirm the subscription before the alarm can notify you

# Alarm: notify if Lambda errors exceed 5 in a 5-minute window
aws cloudwatch put-metric-alarm \
  --alarm-name training-lambda-errors \
  --namespace AWS/Lambda \
  --metric-name Errors \
  --dimensions Name=FunctionName,Value=training-hello \
  --statistic Sum \
  --period 300 \
  --evaluation-periods 1 \
  --threshold 5 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:training-alerts
```

Check current alarm state any time:

```bash
aws cloudwatch describe-alarms --alarm-names training-lambda-errors \
  --query "MetricAlarms[0].StateValue"
# "OK"
```

A common, very useful first alarm on an EC2 instance is high CPU:

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name training-ec2-high-cpu \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=i-0123456789abcdef0 \
  --statistic Average \
  --period 300 \
  --evaluation-periods 2 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:training-alerts
```

## CloudWatch Logs Insights: querying logs like a database

For anything beyond "tail the last few lines," Logs Insights lets you query
log content with a small purpose-built query language:

```bash
aws logs start-query \
  --log-group-name /aws/lambda/training-hello \
  --start-time "$(date -u -v-1H +%s)" \
  --end-time "$(date -u +%s)" \
  --query-string 'fields @timestamp, @message | filter @message like /ERROR/ | sort @timestamp desc | limit 20'
# queryId: abcd-1234-...

aws logs get-query-results --query-id abcd-1234-...
```

## Cheat sheet

| Command | Purpose |
|---|---|
| `aws cloudwatch list-metrics --namespace NS` | Discover what metrics exist. |
| `aws cloudwatch get-metric-statistics ...` | Pull numeric data points for a metric. |
| `aws logs describe-log-groups` | List log groups. |
| `aws logs tail /aws/lambda/NAME --follow` | Stream logs live. |
| `aws logs put-retention-policy --retention-in-days N` | Stop logs from accumulating forever. |
| `aws sns create-topic` / `subscribe` | Create a notification channel for alarms. |
| `aws cloudwatch put-metric-alarm ...` | Create an alarm that fires on a threshold. |
| `aws cloudwatch describe-alarms` | Check current alarm state. |
| `aws logs start-query` / `get-query-results` | Run a Logs Insights query. |

## How It Actually Works

CloudWatch metrics are not sampled continuously in the way a local
monitoring agent might poll — they're **published as discrete data points**
by the service or agent emitting them (EC2's hypervisor emits CPU/network
metrics every 1 or 5 minutes depending on "detailed monitoring"; your own
`put-metric-data` calls create points whenever you call them), and
CloudWatch's storage layer is really a time-series database that
aggregates points landing in the same period into statistics (Average, Sum,
MO, p99, etc.) rather than storing every raw point forever — which is why
older data is only queryable at coarser resolution: CloudWatch automatically
rolls 1-minute data up into 5-minute, then 1-hour aggregates as it ages out,
trading precision for storage cost.

**Alarms** are a state machine, not a continuous check: an alarm evaluates
on a schedule (your period × evaluation-periods), and only transitions
between `OK`, `ALARM`, and `INSUFFICIENT_DATA` when enough consecutive
periods meet the threshold — this deliberate delay (not reacting to a single
spike) is what prevents flapping, but it's also exactly why an alarm can
lag several minutes behind a real, ongoing incident.

Logs work on a completely different backend: CloudWatch Logs ingests
line-oriented events into **log streams** grouped by **log group**, storing
them append-only and indexing them for Logs Insights queries, which is why
Insights queries scan compressed data proportional to the time range you
select rather than doing a live grep — querying a wide time window across a
high-volume log group is a genuine (and billed) full-text scan, not an
instant lookup.

## Exercise

1. Set a 14-day retention policy on the log group for your `training-hello`
   Lambda function.
2. Create an SNS topic, subscribe your own email, and confirm the
   subscription from the confirmation email AWS sends.
3. Create an alarm on that function's `Errors` metric (threshold: greater
   than 0 over a 5-minute period) wired to the SNS topic.
4. Deliberately break the function (e.g. reference an undefined variable),
   invoke it a few times, and confirm you receive an email alert and that
   `describe-alarms` shows `ALARM` state.
5. Fix the function, invoke it successfully a couple of times, and confirm
   the alarm returns to `OK`.
