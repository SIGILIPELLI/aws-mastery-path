# Chaos Engineering & Resilience Testing

Level 3 module 7 designed DR strategies on paper (backup/restore,
pilot light, warm standby, active-active). Chaos engineering is how you
find out whether that design actually works — by deliberately
injecting the failures it's supposed to survive, in a controlled way,
before a real outage does it for you uncontrolled.

## AWS Fault Injection Simulator (FIS)

FIS runs **experiments** — defined sets of fault-injection **actions**
against **targets**, with **stop conditions** that abort automatically
if things go further wrong than expected.

```json
{
  "description": "Terminate one instance in the training-app ASG",
  "targets": {
    "instances-to-terminate": {
      "resourceType": "aws:ec2:instance",
      "selectionMode": "PERCENT(50)",
      "resourceTags": { "Environment": "staging", "App": "training-app" }
    }
  },
  "actions": {
    "terminate-instances": {
      "actionId": "aws:ec2:terminate-instances",
      "targets": { "Instances": "instances-to-terminate" }
    }
  },
  "stopConditions": [
    { "source": "aws:cloudwatch:alarm", "value": "arn:aws:cloudwatch:us-east-1:123456789012:alarm:high-error-rate" }
  ],
  "roleArn": "arn:aws:iam::123456789012:role/FISExperimentRole"
}
```

```bash
aws fis create-experiment-template --cli-input-json file://experiment.json
aws fis start-experiment --experiment-template-id EXT12345678
aws fis get-experiment --id EXP12345678 --query 'state'
# { "status": "completed" }
```

The `stopConditions` entry is what makes this safe to run against real
infrastructure: if the `high-error-rate` CloudWatch alarm fires during
the experiment, FIS halts immediately rather than letting the chaos
run to completion regardless of impact.

## Common experiment types

| Action | Simulates |
|---|---|
| `aws:ec2:terminate-instances` | Sudden instance loss |
| `aws:ec2:stop-instances` | Instance becomes unavailable (not terminated) |
| `aws:ecs:stop-task` | Container task failure |
| `aws:eks:pod-delete` | Pod-level failure in a Kubernetes deployment |
| `aws:network:disrupt-connectivity` | Network partition / latency between AZs |
| `aws:ssm:send-command` (CPU/memory stress via SSM doc) | Resource exhaustion on an instance |

Running `aws:network:disrupt-connectivity` between AZs is a direct test
of the multi-AZ reliability claims made in Level 4 module 1's
Well-Architected review — it either confirms the failover works, or
surfaces the HRI in practice instead of on paper.

## Designing a game day

A structured chaos exercise (a "game day") follows a specific
sequence: state a hypothesis (e.g., "if one AZ's instances all
terminate, the ALB reroutes traffic and error rate stays under 1%
within 2 minutes"), run the minimum experiment to test it, observe
real metrics (via CloudWatch/X-Ray from Level 3 module 8), and record
whether the hypothesis held.

```bash
aws fis list-experiments --query 'experiments[*].{id:id,state:state.status,startTime:startTime}'
```

## Starting small: blast radius control

```json
{
  "targets": {
    "single-instance": {
      "resourceType": "aws:ec2:instance",
      "selectionMode": "COUNT(1)",
      "resourceTags": { "Environment": "staging" }
    }
  }
}
```

`selectionMode: COUNT(1)` limits an experiment's impact to exactly one
resource, regardless of how many match the tag filter — always start
experiments this narrow, in staging, before ever running a
percentage-based or production experiment.

## Gotchas

- **Chaos experiments in production need explicit organizational
  buy-in and a rollback plan** — this is exactly the kind of action
  that should be scheduled, communicated, and reversible; never run an
  untested experiment template against production for the first time.
- **Stop conditions only help if the alarm they reference is actually
  well-tuned** — an experiment guarded by a stop condition tied to a
  noisy or slow-to-trigger alarm doesn't actually protect you; validate
  the alarm's behavior independently first.
- **FIS requires an IAM role with permission to perform the disruptive
  action itself** (e.g., `ec2:TerminateInstances`) — scoping that role
  too broadly turns the experiment framework itself into a risk; scope
  it to the specific resource tags/ARNs the experiment targets.
- **A passed experiment today doesn't mean permanent resilience** —
  infrastructure changes (a new dependency, a changed Auto Scaling
  policy) can silently break a previously-verified failure path; rerun
  key experiments after significant architecture changes, similar to
  the Well-Architected review's periodic re-check.
- **Chaos testing distributed systems can trigger cascading failures
  you didn't intend** — start with single-target, single-AZ blast
  radius, and only widen scope once you've built confidence in both the
  system's resilience and your team's incident response.

## Cheat sheet

| Command | Purpose |
|---|---|
| `aws fis create-experiment-template` | Define an experiment |
| `aws fis start-experiment` | Run it |
| `aws fis get-experiment` | Check status/results |
| `aws fis stop-experiment` | Manually abort a running experiment |
| `aws fis list-experiments` | Review experiment history |

## How It Actually Works

**AWS Fault Injection Service (FIS)** doesn't simulate failures at the
application layer — it injects real faults at the infrastructure level by
calling the same underlying AWS APIs a genuine failure would trigger (or
using SSM Agent-executed OS-level commands on the target instance), such as
actually terminating a percentage of instances in an ASG, actually throttling
network throughput at the ENI level, or actually injecting CPU/memory
pressure via the SSM agent running on the instance — this is a deliberate
design choice: testing your monitoring, alarms, and automated recovery
paths (Multi-AZ failover, ASG replacement, Route 53 health-check failover)
against a *simulated* fault would only prove those systems can detect
simulations, not real failures.

FIS experiments have mandatory **stop conditions** — CloudWatch alarms that,
if triggered during the experiment, cause FIS to immediately halt and (where
supported) roll back the injected fault — implemented as FIS itself
polling the specified alarm's state during the experiment's run, which is
the actual safety mechanism preventing a chaos experiment from cascading
into a genuine, uncontrolled outage: it's not a policy or a suggestion, it's
an automated circuit-breaker built into how the experiment executes.

This connects directly back to the Well-Architected Reliability pillar's
guidance to "test recovery procedures regularly" (module 01 of this level):
a Multi-AZ RDS failover, an ASG health-check replacement, or a Route 53
failover routing policy are all mechanisms with detection thresholds and
cutover logic that, absent deliberate chaos testing, only get exercised
during an actual production incident — FIS's purpose is precisely to
exercise these same code paths (real termination, real throttling) on your
own schedule, with the guardrails above in place, so the first time a
failover mechanism runs isn't during a real outage.

## Exercise

Write an FIS experiment template that terminates exactly one EC2
instance tagged `Environment=staging` in an Auto Scaling group, guarded
by a stop condition on a CloudWatch alarm watching 5xx error rate.
State the hypothesis you're testing before running it, and what
specific CloudWatch/X-Ray evidence (from Level 3 modules 8-9) would
confirm or refute it.
