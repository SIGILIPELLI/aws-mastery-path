# 03 · Route 53 & DNS

Every AWS resource you've built so far is reachable by an AWS-assigned
address — an EC2 public IP, an ALB's `DNSName`, an API Gateway endpoint.
**Route 53** is AWS's DNS service: it lets you put a real, memorable domain
name in front of any of them, and it understands AWS resources well enough
to route directly to them without an extra DNS lookup. This module covers
hosted zones, the record types you'll use most, and pointing a domain at
the ALB from the previous module.

## Core concepts

| Concept | What it is |
|---|---|
| **Hosted zone** | A container for DNS records for one domain (e.g. `example.com`), with its own set of 4 name servers. |
| **Record set** | One DNS entry: a name, a type, and a value (e.g. `www.example.com A 203.0.113.10`). |
| **Alias record** | A Route 53-specific record type that points at an AWS resource (ALB, CloudFront, S3 website) — works at the zone apex, unlike `CNAME`. |
| **TTL** | Time-to-live — how long resolvers cache a record before re-querying. |
| **Routing policy** | How Route 53 chooses among multiple records for the same name — simple, weighted, latency-based, failover, geolocation. |
| **Health check** | A Route 53 probe against an endpoint, usable to drive failover routing. |

## Create a hosted zone

```bash
aws route53 create-hosted-zone \
  --name training.example.com \
  --caller-reference "$(date +%s)"
# HostedZone.Id: /hostedzone/Z1D633PJN98FT9
# NameServers: [ns-123.awsdns-45.com, ns-678.awsdns-90.net, ...]
```

`--caller-reference` just needs to be unique per call (a timestamp works)
so retried requests aren't accidentally treated as duplicates. If this is
a real subdomain of a domain you own elsewhere, you'd add these 4 name
servers as `NS` records at your registrar to delegate `training.example.com`
to this zone — a step this training exercise skips by working within a
zone you already control end-to-end.

## Add an alias record pointing at the ALB

```bash
cat > record-change.json << 'EOF'
{
  "Changes": [
    {
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "app.training.example.com",
        "Type": "A",
        "AliasTarget": {
          "HostedZoneId": "Z35SXDOTRQ7X7K",
          "DNSName": "training-alb-123456789.us-east-1.elb.amazonaws.com",
          "EvaluateTargetHealth": true
        }
      }
    }
  ]
}
EOF

aws route53 change-resource-record-sets \
  --hosted-zone-id /hostedzone/Z1D633PJN98FT9 \
  --change-batch file://record-change.json
# ChangeInfo.Status: PENDING
```

`Z35SXDOTRQ7X7K` here is the ALB's own fixed **hosted zone ID** (every ELB
type has one, specific to its region — found via `aws elbv2
describe-load-balancers --query "LoadBalancers[0].CanonicalHostedZoneId"`),
not the zone you just created. An alias `A` record has no TTL of its own
and, unlike a `CNAME`, is allowed at a zone's apex (`example.com` itself,
with no subdomain) — this is the main reason to prefer alias records over
`CNAME` for anything pointing at an AWS resource.

## Wait for the change and verify

```bash
aws route53 get-change --id /change/C1A2B3C4D5E6F
# Status: INSYNC

aws route53 list-resource-record-sets \
  --hosted-zone-id /hostedzone/Z1D633PJN98FT9 \
  --query "ResourceRecordSets[].[Name,Type]" --output table

dig app.training.example.com
nslookup app.training.example.com
```

Propagation across the public DNS system can still lag behind
`INSYNC` — resolvers and local caches may hold an old (or absent) answer
for a while, governed by the record's TTL (or, for a first-ever lookup,
however long the previous negative answer was cached).

## Weighted routing (canary traffic split)

```bash
cat > weighted-blue.json << 'EOF'
{
  "Changes": [{
    "Action": "UPSERT",
    "ResourceRecordSet": {
      "Name": "app.training.example.com",
      "Type": "A",
      "SetIdentifier": "blue",
      "Weight": 90,
      "AliasTarget": {
        "HostedZoneId": "Z35SXDOTRQ7X7K",
        "DNSName": "training-alb-123456789.us-east-1.elb.amazonaws.com",
        "EvaluateTargetHealth": true
      }
    }
  }]
}
EOF
aws route53 change-resource-record-sets \
  --hosted-zone-id /hostedzone/Z1D633PJN98FT9 --change-batch file://weighted-blue.json
```

A second record with the same `Name`/`Type` but `SetIdentifier: green` and
`Weight: 10` sends roughly 10% of resolutions to a second target (e.g. a
canary ALB) — useful for gradually shifting traffic to a new environment
without touching application infrastructure.

## Failover with a health check

```bash
aws route53 create-health-check \
  --caller-reference "$(date +%s)" \
  --health-check-config '{
    "Type": "HTTPS",
    "FullyQualifiedDomainName": "app.training.example.com",
    "Port": 443,
    "ResourcePath": "/health",
    "RequestInterval": 30,
    "FailureThreshold": 3
  }'
# HealthCheck.Id: abcd1234-ab12-cd34-ef56-abcdef123456
```

Attach `HealthCheckId` plus `Failover: PRIMARY` to your main record and a
second record with `Failover: SECONDARY` pointing at a standby — Route 53
stops answering with the primary's address once its health check fails 3
consecutive times, and resumes once it recovers.

## Registering a domain (if you don't already own one)

```bash
aws route53domains check-domain-availability --domain-name training-app-example.com
aws route53domains register-domain --cli-input-json file://domain-registration.json
```

`route53domains` is a separate API (and console area) from `route53`
itself — it handles the ICANN registration/renewal side, while the
`route53` hosted zone handles DNS answers for a domain regardless of where
it's registered.

!!! warning "TTL is a tradeoff between propagation speed and query volume/cost"
    A low TTL (e.g. 60s) means changes (including failover) propagate
    fast, but resolvers re-query Route 53 far more often, at proportionally
    higher query cost. A high TTL is cheaper and fine for stable records,
    but a mistake — or a needed failover — takes that much longer to reach
    every caching resolver.

## Cheat sheet

| Command | Purpose |
|---|---|
| `aws route53 create-hosted-zone --name DOMAIN --caller-reference X` | Create a DNS zone for a domain. |
| `aws route53 change-resource-record-sets --change-batch file://F` | Create/update/delete records (UPSERT/CREATE/DELETE). |
| `aws route53 get-change --id ID` | Check whether a record change has propagated to Route 53's servers. |
| `aws route53 list-resource-record-sets --hosted-zone-id Z` | List all records in a zone. |
| `aws route53 create-health-check` | Define an endpoint probe for failover routing. |
| `aws route53domains check-domain-availability` | Check if a domain name can be registered. |
| `dig NAME` / `nslookup NAME` | Query DNS directly to verify what resolvers actually see. |

## How It Actually Works

Route 53 is a globally-distributed **authoritative DNS** service — when you
create a hosted zone, AWS assigns it four name servers spread across
different top-level DNS provider infrastructures (a deliberate diversity
choice, so a single provider's outage doesn't take your DNS with it), and
propagates your records to Route 53's edge network of DNS resolvers
worldwide via an internal, low-latency replication system, not the public
internet's DNS propagation path — this is why Route 53 changes typically
apply within seconds, unlike the "24–48 hours" folklore associated with DNS
in general (which really refers to caching by *resolvers downstream* of
Route 53, governed by the TTL you set).

An **alias record** is Route 53-specific: unlike a CNAME (which can't
coexist with other records at the zone apex per the DNS spec, and always
costs a client an extra lookup), an alias record is resolved *inside* Route
53's own infrastructure — at query time, Route 53 looks up the current IP(s)
behind the target AWS resource (an ALB, CloudFront distribution, S3
website endpoint) and returns those directly, for free, with no extra
client-side hop, and automatically stays correct if the target's underlying
IPs change, which they routinely do behind a load balancer.

**Routing policies** (weighted, latency-based, failover, geolocation) are
implemented by Route 53's resolvers making a per-query decision using data
they hold locally: latency-based routing consults a static latency table
between AWS's own edge locations and each region (not a live probe of the
querying client), while health-check-based failover routing depends on
Route 53's own health checkers (a separate global fleet) polling your
endpoint and updating the record's eligibility within seconds of a failure
being confirmed from multiple checker locations, to avoid flapping caused by
one checker's transient network blip.

## Exercise

1. Create a hosted zone for a subdomain you control (or a dummy domain for
   practice), and note its 4 name servers.
2. Add an alias `A` record pointing `app.<zone>` at the ALB from module 2,
   with `EvaluateTargetHealth: true`.
3. Confirm with `dig` that the name resolves, and that visiting it in a
   browser reaches the ALB.
4. Create a Route 53 health check against a real path on your app, and
   attach `PRIMARY`/`SECONDARY` failover records so a second target group
   (or a static "sorry" page) takes over if the primary starts failing.
5. Force a health check failure (e.g. stop all instances behind the target
   group) and confirm `dig` starts returning the secondary's address once
   the failure threshold is hit; restore the primary afterward.
6. Delete the extra records, the health check, and the hosted zone when
   done (a non-default hosted zone bills a small monthly fee per zone).
