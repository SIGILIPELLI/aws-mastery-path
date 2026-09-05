# 04 · SNS & SQS Messaging

So far every service has talked to another synchronously — a caller waits
for a direct response. Messaging services decouple that: a producer sends
a message and moves on, and one or more consumers process it whenever
they're ready. **SNS** (Simple Notification Service) is pub/sub — one
message fans out to many subscribers. **SQS** (Simple Queue Service) is a
point-to-point queue — one message is processed by exactly one consumer
(or, in a FIFO queue, one message group). Combining them (SNS → multiple
SQS queues) is the standard **fan-out** pattern.

## Core concepts

| Concept | What it is |
|---|---|
| **Topic (SNS)** | A named channel producers publish to; each subscriber gets its own copy of every message. |
| **Queue (SQS)** | A durable buffer of messages; each message is delivered to (and processed by) one consumer. |
| **Subscription** | A binding from an SNS topic to an endpoint — SQS queue, Lambda, email, HTTP(S). |
| **Visibility timeout** | How long a received-but-not-yet-deleted SQS message is hidden from other consumers, to avoid double-processing. |
| **Dead-letter queue (DLQ)** | A separate queue messages move to after failing processing too many times, so they don't block the queue forever. |
| **FIFO queue** | An ordered, exactly-once-processing queue variant (`.fifo` name suffix), vs. the default "standard" queue (at-least-once, best-effort order). |

## Create an SNS topic

```bash
aws sns create-topic --name training-orders
# TopicArn: arn:aws:sns:us-east-1:123456789012:training-orders
```

## Create SQS queues (main + dead-letter)

```bash
aws sqs create-queue --queue-name training-orders-dlq
# QueueUrl: https://sqs.us-east-1.amazonaws.com/123456789012/training-orders-dlq

aws sqs get-queue-attributes \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456789012/training-orders-dlq \
  --attribute-names QueueArn
# QueueArn: arn:aws:sqs:us-east-1:123456789012:training-orders-dlq

aws sqs create-queue \
  --queue-name training-orders-queue \
  --attributes '{
    "VisibilityTimeout": "30",
    "RedrivePolicy": "{\"deadLetterTargetArn\":\"arn:aws:sqs:us-east-1:123456789012:training-orders-dlq\",\"maxReceiveCount\":\"5\"}"
  }'
# QueueUrl: https://sqs.us-east-1.amazonaws.com/123456789012/training-orders-queue
```

`maxReceiveCount: 5` means a message that's received and left unprocessed
(visibility timeout expires without a delete) 5 times gets moved to the
DLQ instead of retried forever — protecting the rest of the queue from one
poison message.

## Let SNS deliver into the SQS queue

Subscribing SQS to SNS needs two things: the subscription itself, and a
queue policy granting the SNS topic permission to send to it.

```bash
aws sqs get-queue-attributes \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456789012/training-orders-queue \
  --attribute-names QueueArn
# QueueArn: arn:aws:sqs:us-east-1:123456789012:training-orders-queue

cat > sqs-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "sns.amazonaws.com" },
    "Action": "sqs:SendMessage",
    "Resource": "arn:aws:sqs:us-east-1:123456789012:training-orders-queue",
    "Condition": {
      "ArnEquals": { "aws:SourceArn": "arn:aws:sns:us-east-1:123456789012:training-orders" }
    }
  }]
}
EOF

aws sqs set-queue-attributes \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456789012/training-orders-queue \
  --attributes "{\"Policy\": \"$(cat sqs-policy.json | tr -d '\n' | sed 's/"/\\"/g')\"}"

aws sns subscribe \
  --topic-arn arn:aws:sns:us-east-1:123456789012:training-orders \
  --protocol sqs \
  --notification-endpoint arn:aws:sqs:us-east-1:123456789012:training-orders-queue
# SubscriptionArn: arn:aws:sns:us-east-1:123456789012:training-orders:...
```

Forgetting the queue policy is the single most common SNS→SQS mistake:
the subscription looks fine, but every publish silently fails to reach
the queue because SNS isn't authorized to call `SendMessage` on it.

## Publish and consume

```bash
aws sns publish \
  --topic-arn arn:aws:sns:us-east-1:123456789012:training-orders \
  --message '{"orderId": "1001", "total": 42.50}' \
  --message-attributes '{"eventType": {"DataType": "String", "StringValue": "OrderCreated"}}'
# MessageId: 5f9e8d7c-...

aws sqs receive-message \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456789012/training-orders-queue \
  --wait-time-seconds 10 --max-number-of-messages 1
# Messages[0].Body contains the SNS envelope (Message, MessageId, TopicArn, ...)
# Messages[0].ReceiptHandle: AQEB3O5nD...

# After successfully processing, delete it so it isn't redelivered
aws sqs delete-message \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456789012/training-orders-queue \
  --receipt-handle "AQEB3O5nD..."
```

`--wait-time-seconds 10` enables **long polling** — the call blocks up to
10 seconds waiting for a message instead of returning empty immediately,
which is both cheaper (fewer empty API calls) and lower-latency than
polling in a tight loop.

## Filtering: only route matching messages to a subscriber

```bash
aws sns set-subscription-attributes \
  --subscription-arn arn:aws:sns:us-east-1:123456789012:training-orders:abcd-1234 \
  --attribute-name FilterPolicy \
  --attribute-value '{"eventType": ["OrderCreated", "OrderCancelled"]}'
```

With a filter policy attached, this subscriber only receives messages
whose `eventType` message attribute matches — a second subscriber (e.g. an
analytics queue) can have no filter and receive everything, giving you
selective fan-out from one topic.

## Standard vs. FIFO queues

| | Standard queue | FIFO queue (`.fifo`) |
|---|---|---|
| Ordering | Best-effort, not guaranteed | Strict, per message group |
| Delivery | At-least-once (duplicates possible) | Exactly-once processing |
| Throughput | Nearly unlimited | Up to 3,000 msg/sec per API action (with batching) |
| Name requirement | Any | Must end in `.fifo` |

```bash
aws sqs create-queue \
  --queue-name training-orders.fifo \
  --attributes '{"FifoQueue": "true", "ContentBasedDeduplication": "true"}'
```

Default to standard queues unless you specifically need ordering or
exactly-once semantics — FIFO's throughput ceiling and the extra
`MessageGroupId`/`MessageDeduplicationId` parameters it requires are real
costs to justify.

!!! warning "At-least-once delivery means your consumer must be idempotent"
    A standard SQS queue (and SNS delivery to it) can redeliver a message
    more than once — a network blip after processing but before
    `delete-message` looks identical to "never processed" from the queue's
    point of view. Design consumers so processing the same message twice
    (e.g. by `orderId`) is harmless.

## Cheat sheet

| Command | Purpose |
|---|---|
| `aws sns create-topic --name NAME` | Create a pub/sub topic. |
| `aws sqs create-queue --attributes '{"RedrivePolicy": ...}'` | Create a queue with a dead-letter policy. |
| `aws sns subscribe --protocol sqs --notification-endpoint QUEUE_ARN` | Wire a queue to receive topic messages. |
| `aws sqs set-queue-attributes --attributes '{"Policy": ...}'` | Grant SNS permission to send to the queue. |
| `aws sns publish --message MSG --message-attributes ATTRS` | Publish a message to a topic. |
| `aws sqs receive-message --wait-time-seconds N` | Long-poll for messages. |
| `aws sqs delete-message --receipt-handle H` | Acknowledge successful processing. |
| `aws sns set-subscription-attributes --attribute-name FilterPolicy` | Restrict which messages a subscriber receives. |

## How It Actually Works

SQS achieves durability by writing each message to **multiple servers
across the queue's storage backend** before returning success to the
sender — the queue is not a single in-memory buffer. A standard queue's
famous "at-least-once, best-effort ordering" behavior comes directly from
this distributed design: because messages are redundantly stored, consuming
a message doesn't delete it; it makes it **invisible** for the duration of
your visibility timeout, and only a subsequent explicit `DeleteMessage` call
removes it from the underlying store. If your consumer crashes before
calling delete, the message reappears after the timeout — this is a
deliberate design (guaranteeing delivery over guaranteeing exactly-once) and
is why idempotent message processing is a hard requirement, not a
nice-to-have, for standard queues.

**FIFO queues** trade some throughput for the additional guarantee of
ordering *within a message group*, implemented by internally partitioning
message storage per group ID and using a message-deduplication ID (or
content hash) to reject duplicate sends within a 5-minute window — this
dedup is done at write time against a hash index, not by inspecting message
bodies at consume time.

SNS is a **fan-out publish/subscribe** system: publishing one message
triggers SNS's delivery workers to push independently to every current
subscriber (SQS queues, Lambda functions, HTTP endpoints, email) in
parallel, each with its own retry policy — this is why an SNS→SQS fan-out
pattern is so common for decoupling: SNS handles the "notify everyone who
cares" broadcast problem, while each SQS queue on the receiving end
independently handles the "buffer and retry until my specific consumer is
ready" problem, and a slow or failing subscriber can't block delivery to the
others.

## Exercise

1. Create an SNS topic and two SQS queues: one main queue with a DLQ
   (`maxReceiveCount: 5`), one plain queue with no DLQ.
2. Subscribe both queues to the topic, with correct queue policies granting
   SNS send permission on each.
3. Add a `FilterPolicy` to the main queue's subscription so it only
   receives messages with `eventType: OrderCreated`; leave the second
   queue unfiltered.
4. Publish 3 messages with different `eventType` attributes, and confirm
   via `receive-message` on each queue that filtering worked as expected.
5. Simulate a poison message: receive a message from the main queue but
   never delete it, and confirm (via repeated receives past the visibility
   timeout, or by checking `ApproximateReceiveCount`) that after 5 failed
   receives it appears in the DLQ instead.
6. Delete the subscriptions, queues, and topic when finished.
