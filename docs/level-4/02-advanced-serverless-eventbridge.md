# Advanced Serverless (EventBridge at Scale)

Level 1 used EventBridge (or its predecessor CloudWatch Events) for
simple scheduled triggers. At scale, EventBridge becomes the backbone
of event-driven architecture: many producers publish events without
knowing who consumes them, and many consumers subscribe by pattern
without knowing who produced them.

## Custom event buses

The default bus receives AWS service events; production systems create
**custom buses** per domain so application events don't mix with AWS
service noise, and so cross-account/cross-team access can be scoped
per bus.

```bash
aws events create-event-bus --name orders-bus

aws events put-permission \
  --event-bus-name orders-bus \
  --action events:PutEvents \
  --principal 222222222222 \
  --statement-id AllowShippingAccount
```

`put-permission` lets another AWS account publish to your bus without
sharing credentials — the shipping team's account 222222222222 can
`PutEvents` on `orders-bus` using only its own IAM role.

## Publishing and routing events

```bash
aws events put-events --entries '[{
  "Source": "orders.api",
  "DetailType": "OrderPlaced",
  "EventBusName": "orders-bus",
  "Detail": "{\"orderId\":\"ord-8842\",\"amount\":49.99,\"tier\":\"enterprise\"}"
}]'
```

```json
{
  "Name": "route-enterprise-orders",
  "EventPattern": {
    "source": ["orders.api"],
    "detail-type": ["OrderPlaced"],
    "detail": { "tier": ["enterprise"] }
  }
}
```

```bash
aws events put-rule \
  --name route-enterprise-orders \
  --event-bus-name orders-bus \
  --event-pattern file://pattern.json

aws events put-targets \
  --rule route-enterprise-orders \
  --event-bus-name orders-bus \
  --targets '[{"Id":"1","Arn":"arn:aws:lambda:us-east-1:123456789012:function:priorityFulfillment"}]'
```

The pattern-matching is content-based: `detail.tier` filters at the
bus level, so `priorityFulfillment` only fires for enterprise orders
without any code checking the tier itself.

## Schema registry

EventBridge can infer and store a **schema** for events flowing through
a bus, generating strongly-typed code bindings for consumers.

```bash
aws schemas create-registry --registry-name orders-schemas

aws schemas search-schema \
  --registry-name orders-schemas \
  --keywords OrderPlaced

aws schemas get-code-binding-source \
  --registry-name orders-schemas \
  --schema-name orders.api@OrderPlaced \
  --language Python36
```

This turns "what fields does an `OrderPlaced` event have" from tribal
knowledge into a queryable, versioned contract between teams.

## Dead-letter queues and retry policy

Targets can fail (throttling, downstream errors) — without a DLQ, a
failed delivery is retried per the rule's policy and then silently
dropped.

```bash
aws events put-targets \
  --rule route-enterprise-orders \
  --event-bus-name orders-bus \
  --targets '[{
    "Id": "1",
    "Arn": "arn:aws:lambda:us-east-1:123456789012:function:priorityFulfillment",
    "RetryPolicy": { "MaximumRetryAttempts": 3, "MaximumEventAgeInSeconds": 3600 },
    "DeadLetterConfig": { "Arn": "arn:aws:sqs:us-east-1:123456789012:orders-dlq" }
  }]'
```

## Archive and replay

```bash
aws events create-archive \
  --archive-name orders-archive \
  --event-source-arn arn:aws:events:us-east-1:123456789012:event-bus/orders-bus \
  --retention-days 90

aws events start-replay \
  --replay-name replay-2026-08-24-incident \
  --event-source-arn arn:aws:events:us-east-1:123456789012:event-bus/orders-bus \
  --event-start-time 2026-08-24T00:00:00Z \
  --event-end-time 2026-08-24T06:00:00Z \
  --destination '{"Arn":"arn:aws:events:us-east-1:123456789012:event-bus/orders-bus"}'
```

Archive + replay is how you recover from a downstream consumer bug
without asking every producer to re-send events — replay re-delivers
exactly what was originally published within the time window.

## Gotchas

- **EventBridge delivery is at-least-once** — consumers must be
  idempotent (e.g., dedupe on `orderId`), since the same event can be
  delivered twice under retry or replay scenarios.
- **Event pattern matching is exact on structure** — a producer that
  changes `detail.tier` to `detail.customerTier` breaks every rule
  silently (no error, matching rules just stop firing); schema
  registry helps catch this at the contract level, not runtime.
- **Cross-account buses need permissions on both sides** — the
  publisher needs `events:PutEvents` permission via `put-permission` on
  the target bus, and IAM permission in its own account to call
  EventBridge at all.
- **Replays re-trigger every rule matching the pattern for that
  window**, including ones unrelated to the original incident — scope
  the replay's event pattern narrowly or you'll re-process unrelated
  events too.
- **Archives cost storage over the retention period** — a 90-day
  archive on a high-volume bus is a real, ongoing cost; set retention
  to what you actually need for replay/audit, not indefinitely.

## Cheat sheet

| Command | Purpose |
|---|---|
| `aws events create-event-bus` | Create a custom bus |
| `aws events put-events` | Publish an event |
| `aws events put-rule` / `put-targets` | Route events by pattern |
| `aws schemas create-registry` | Track event schemas |
| `aws events create-archive` | Enable replay capability |
| `aws events start-replay` | Re-deliver past events |

## How It Actually Works

EventBridge is architecturally a **publish/subscribe event bus** built
around **rules** that pattern-match against event JSON — when an event is
published (either by an AWS service automatically, or your own
`PutEvents` call), EventBridge evaluates every rule on the target event
bus against that event's structure, and for each matching rule, invokes
every configured target *independently and in parallel*, similar in spirit
to SNS fan-out but with far richer content-based routing: a rule can match
on nested JSON fields, numeric ranges, and prefixes rather than just a
topic name.

Because target invocation is decoupled per-rule, EventBridge maintains its
own **retry policy with exponential backoff and dead-letter queue support**
per target — a failure delivering to one target (say, a Lambda function
throttling) has zero effect on delivery to any other target matched by the
same or a different rule; each target's retry state is tracked
independently by EventBridge's delivery subsystem, not by the publishing
service.

**Schema Registry** and its "schema discovery" feature work by EventBridge
passively sampling a percentage of events flowing through a bus and running
structural inference over them to generate an OpenAPI/JSONSchema definition
— it's not a strict validation gate by default (events don't need to match
a registered schema to be delivered), which is a deliberate trade-off:
EventBridge prioritizes not silently dropping events over enforcing schema
compliance, leaving strict validation to be opted into at the consumer
level if you need it.

## Exercise

Create a custom event bus, publish three `OrderPlaced` events with
different `tier` values, and write an event pattern that routes only
`tier: "enterprise"` events to a DLQ-backed Lambda target. Then create
a 7-day archive on the bus and replay the last hour of events into a
second rule you add afterward — confirming replay reaches rules that
didn't exist when the events were first published.
