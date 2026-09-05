# Multi-Region & Disaster Recovery

Everything so far ran in one region. If that region has an outage —
rare, but it happens — a single-region architecture is fully down.
Disaster recovery (DR) planning trades cost against recovery speed
across four standard strategies.

## The four DR strategies

| Strategy | RTO (recovery time) | RPO (data loss) | Relative cost |
|---|---|---|---|
| Backup & Restore | Hours | Minutes-hours since last backup | $ |
| Pilot Light | Minutes-hours | Minutes | $$ |
| Warm Standby | Minutes | Seconds-minutes | $$$ |
| Multi-Site Active-Active | Near-zero | Near-zero | $$$$ |

**RTO** (Recovery Time Objective) is how long you can tolerate being
down. **RPO** (Recovery Point Objective) is how much data you can
tolerate losing. Pick a strategy by working backward from business
requirements, not by defaulting to the most expensive option.

## Backup & Restore

Cheapest: back up data to another region, keep infrastructure-as-code
ready, and only provision compute in the DR region after a declared
disaster.

```bash
# Cross-region S3 replication for backups
aws s3api put-bucket-replication \
  --bucket training-app-primary \
  --replication-configuration file://replication.json

# RDS: automated snapshots copied cross-region
aws rds copy-db-snapshot \
  --source-db-snapshot-identifier arn:aws:rds:us-east-1:123456789012:snapshot:prod-snap-2026-08-20 \
  --target-db-snapshot-identifier prod-snap-dr-copy \
  --source-region us-east-1 \
  --region us-west-2
```

Recovery means running your CloudFormation/Terraform templates in the
DR region and restoring from the latest snapshot — hours of RTO, but
minimal ongoing cost since nothing runs until needed.

## Pilot Light

A minimal version of the core system (typically just the database,
kept in sync) runs continuously in the DR region; compute is scaled to
zero and only started up during a failover.

```bash
# Keep an RDS read replica warm in the DR region
aws rds create-db-instance-read-replica \
  --db-instance-identifier prod-db-replica-west \
  --source-db-instance-identifier arn:aws:rds:us-east-1:123456789012:db:prod-db \
  --region us-west-2

# On failover: promote the replica and scale up compute
aws rds promote-read-replica --db-instance-identifier prod-db-replica-west --region us-west-2
aws autoscaling update-auto-scaling-group \
  --auto-scaling-group-name prod-asg-west --min-size 4 --desired-capacity 4 --region us-west-2
```

## Warm Standby

A scaled-down but fully functional copy runs continuously in the DR
region — it can serve some traffic even before failover, and scales up
fast when needed.

## Multi-Site Active-Active

Both regions run full production capacity simultaneously behind global
traffic routing:

```bash
aws route53 create-health-check \
  --caller-reference primary-region-hc \
  --health-check-config Type=HTTPS,ResourcePath=/health,FullyQualifiedDomainName=app-us-east-1.example.com

aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890ABC \
  --change-batch file://failover-routing.json
```

```json
{
  "Changes": [{
    "Action": "UPSERT",
    "ResourceRecordSet": {
      "Name": "app.example.com",
      "Type": "A",
      "SetIdentifier": "primary",
      "Failover": "PRIMARY",
      "HealthCheckId": "abc12345-6789-def0-1234-56789abcdef0",
      "AliasTarget": { "HostedZoneId": "Z35SXDOTRQ7X7K", "DNSName": "alb-us-east-1.elb.amazonaws.com", "EvaluateTargetHealth": true }
    }
  }]
}
```

Route 53 **failover routing policy** automatically stops sending
traffic to the primary once its health check fails, shifting to the
secondary record — no manual DNS change needed during an incident.

## Gotchas

- **RTO/RPO numbers are only as good as your last DR test** — a pilot
  light architecture you've never actually failed over to is a guess,
  not a plan. Schedule regular game-day failover drills.
- **Cross-region replication has lag** — RDS read replicas and S3
  cross-region replication are asynchronous; a disaster in the middle
  of replication loses whatever hadn't synced yet, which is exactly
  what RPO measures.
- **IAM, KMS keys, and some services are regional** — a DR plan that
  assumes "just run the same template in another region" breaks if
  it references KMS key ARNs or IAM role ARNs by region-specific
  values that don't exist there yet.
- **Route 53 health checks add cost per check and can flap** on
  transient network blips — tune failure thresholds so a temporary
  hiccup doesn't trigger a full regional failover.
- **Active-active requires your data layer to handle multi-region
  writes** (e.g., DynamoDB Global Tables, Aurora Global Database) —
  bolting active-active onto a single-region RDS primary just means
  one region silently has stale data.

## Cheat sheet

| Task | Command |
|---|---|
| Cross-region snapshot copy | `aws rds copy-db-snapshot --source-region ...` |
| Cross-region S3 replication | `aws s3api put-bucket-replication` |
| Create read replica in another region | `aws rds create-db-instance-read-replica` |
| Promote replica during failover | `aws rds promote-read-replica` |
| DNS failover | `aws route53 change-resource-record-sets` with `Failover` policy |
| Health check | `aws route53 create-health-check` |

## How It Actually Works

Cross-region replication features (S3 CRR, RDS cross-region read replicas,
DynamoDB Global Tables) all share the same underlying constraint: regions
are separate control-plane and data-plane deployments connected only by
AWS's private backbone network, so any cross-region replication is
necessarily **asynchronous** — there is no synchronous cross-region write
path in AWS because the physical distance alone (hundreds to thousands of
miles) makes synchronous replication's round-trip latency unacceptable for
almost any workload. This is the mechanical reason every multi-region DR
strategy has to reckon with **RPO** (how much data you can afford to lose,
bounded by replication lag) as a real, physics-driven number, not just a
policy choice.

DynamoDB Global Tables implement multi-region writes via **multi-master
replication with last-writer-wins conflict resolution**: each region can
accept writes independently, and DynamoDB's internal replication streams
propagate every write to all other participating regions' tables,
resolving any conflicting writes to the same item using each write's
internal timestamp — meaning two near-simultaneous writes to the same key
in different regions can silently have one discarded, which is a
fundamentally different consistency model than a single-region table
offers and needs to be designed around at the application level.

DNS-based failover (Route 53 health checks flipping a failover routing
policy) is a control-plane action layered on top of whatever replication
mechanism your data tier uses — Route 53 detecting an unhealthy primary and
switching authoritative answers to the secondary region only redirects
*new* DNS lookups; clients holding a cached resolution (bounded by your
record's TTL) keep hitting the failed region until that TTL expires, which
is why RTO in a DNS-failover design is never zero even after the failover
itself completes instantly.

## Exercise

Given a single-region web app (ALB + ASG + RDS) with an RTO requirement
of 30 minutes and an RPO of 5 minutes, choose the cheapest DR strategy
from the table that meets both, and list the concrete AWS resources
you'd provision in the secondary region to satisfy it.
