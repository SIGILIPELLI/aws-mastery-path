# 05 · DynamoDB Deep Dive

The Level 1 capstone used DynamoDB with a single-attribute key just to
store and fetch notes by ID. That barely scratches what DynamoDB is built
for. This module covers **composite keys**, **secondary indexes** (the main
tool for querying by something other than the primary key), **capacity
modes**, and **streams** — the concepts you need to actually model an
application's data in DynamoDB rather than just using it as a key-value
store.

## Core concepts

| Concept | What it is |
|---|---|
| **Partition key** | The attribute DynamoDB hashes to decide which physical partition stores an item. |
| **Sort key** | An optional second key attribute; items sharing a partition key are ordered by it, enabling range queries. |
| **GSI** (Global Secondary Index) | An alternate partition/sort key over the same table, queryable independently, with its own eventually-consistent copy of the data. |
| **LSI** (Local Secondary Index) | An alternate sort key using the *same* partition key, created only at table creation, supporting strongly consistent reads. |
| **Capacity mode** | `PAY_PER_REQUEST` (on-demand, pay per read/write) vs. `PROVISIONED` (reserve RCU/WCU capacity, cheaper at steady high volume). |
| **Stream** | An ordered, near-real-time log of item-level changes a table can emit, consumable by Lambda or Kinesis. |

## Composite key table design

```bash
aws dynamodb create-table \
  --table-name training-orders \
  --attribute-definitions \
      AttributeName=customerId,AttributeType=S \
      AttributeName=orderId,AttributeType=S \
      AttributeName=orderStatus,AttributeType=S \
      AttributeName=createdAt,AttributeType=S \
  --key-schema \
      AttributeName=customerId,KeyType=HASH \
      AttributeName=orderId,KeyType=RANGE \
  --billing-mode PAY_PER_REQUEST \
  --global-secondary-indexes '[{
    "IndexName": "byStatusAndDate",
    "KeySchema": [
      {"AttributeName": "orderStatus", "KeyType": "HASH"},
      {"AttributeName": "createdAt", "KeyType": "RANGE"}
    ],
    "Projection": {"ProjectionType": "ALL"}
  }]'
```

`customerId` (partition) + `orderId` (sort) means "all orders for one
customer" is a single efficient query — items sharing a partition key are
stored together and range-queryable by sort key. Only attributes actually
declared in `--attribute-definitions` can be used as a key or index key;
every other attribute (e.g. `total`, `items`) is schemaless and just goes
in the item.

## Writing and querying items

```bash
aws dynamodb put-item \
  --table-name training-orders \
  --item '{
    "customerId": {"S": "cust#42"},
    "orderId": {"S": "order#1001"},
    "orderStatus": {"S": "PLACED"},
    "createdAt": {"S": "2026-08-01T10:00:00Z"},
    "total": {"N": "42.50"}
  }'

# All orders for one customer (uses the base table's key schema directly)
aws dynamodb query \
  --table-name training-orders \
  --key-condition-expression "customerId = :c" \
  --expression-attribute-values '{":c": {"S": "cust#42"}}'

# All PLACED orders across all customers, newest first (uses the GSI)
aws dynamodb query \
  --table-name training-orders \
  --index-name byStatusAndDate \
  --key-condition-expression "orderStatus = :s" \
  --expression-attribute-values '{":s": {"S": "PLACED"}}' \
  --scan-index-forward false
```

Without the GSI, "all orders in PLACED status" would require a full-table
**scan** (reading every item and filtering) — the GSI turns it into an
efficient, targeted `query`. Designing GSIs around your app's actual
access patterns, before you need them, is the core skill of DynamoDB data
modeling.

## Batch operations

```bash
aws dynamodb batch-write-item --request-items file://batch-orders.json
# batch-orders.json: {"training-orders": [{"PutRequest": {"Item": {...}}}, ...]}

aws dynamodb update-item \
  --table-name training-orders \
  --key '{"customerId": {"S": "cust#42"}, "orderId": {"S": "order#1001"}}' \
  --update-expression "SET orderStatus = :s" \
  --expression-attribute-values '{":s": {"S": "SHIPPED"}}' \
  --return-values ALL_NEW
```

`batch-write-item` handles up to 25 items per call and does **not**
support conditional expressions — use individual `put-item`/`update-item`
calls when you need "only if this doesn't already exist" logic.

## Consistency: eventually consistent by default

```bash
# Default read — eventually consistent, cheaper
aws dynamodb get-item \
  --table-name training-orders \
  --key '{"customerId": {"S": "cust#42"}, "orderId": {"S": "order#1001"}}'

# Strongly consistent — guarantees you see the latest write, costs 2x the RCU
aws dynamodb get-item \
  --table-name training-orders \
  --key '{"customerId": {"S": "cust#42"}, "orderId": {"S": "order#1001"}}' \
  --consistent-read
```

**GSI queries are always eventually consistent** — there is no
`--consistent-read` option for a GSI, because the index is itself an
asynchronously-updated copy of the base table. A write followed
immediately by a GSI query can, rarely, not yet reflect that write; the
base table (or an LSI) supports strongly consistent reads because it's the
authoritative copy.

## Switching capacity modes

```bash
aws dynamodb update-table \
  --table-name training-orders \
  --billing-mode PROVISIONED \
  --provisioned-throughput ReadCapacityUnits=5,WriteCapacityUnits=5

aws dynamodb update-table \
  --table-name training-orders \
  --billing-mode PAY_PER_REQUEST
```

`PAY_PER_REQUEST` is the right default for unpredictable or low/spiky
traffic (including everything in this course) — you pay per read/write
with no capacity to plan. `PROVISIONED` (optionally with auto scaling on
top) becomes cheaper once traffic is steady and high enough that reserved
capacity costs less than paying per-request for the same volume.

## Enabling streams and TTL

```bash
aws dynamodb update-table \
  --table-name training-orders \
  --stream-specification StreamEnabled=true,StreamViewType=NEW_AND_OLD_IMAGES

aws dynamodb update-time-to-live \
  --table-name training-orders \
  --time-to-live-specification "Enabled=true,AttributeName=expiresAt"
```

A stream (consumed by a Lambda trigger, similar to module 1's ECS trigger
wiring) can react to every insert/update/delete — e.g. sending an SNS
notification (previous module) whenever `orderStatus` changes. TTL
auto-deletes items once their `expiresAt` (a Unix epoch number attribute)
passes — deletion happens within the following 48 hours typically, not
instantly, and doesn't consume write capacity.

!!! warning "A poorly chosen partition key creates a hot partition"
    DynamoDB spreads throughput across partitions based on the partition
    key's distribution. A key like `orderStatus` alone (few distinct
    values, uneven access) concentrates traffic on very few partitions and
    throttles even though the table's overall capacity looks sufficient.
    Prefer high-cardinality keys (like `customerId`) for the base table,
    and push low-cardinality access patterns into a GSI instead.

## Cheat sheet

| Command | Purpose |
|---|---|
| `aws dynamodb create-table --key-schema HASH+RANGE --global-secondary-indexes [...]` | Create a table with a composite key and a GSI. |
| `aws dynamodb query --key-condition-expression ...` | Efficient, key-based read (base table or an index). |
| `aws dynamodb query --index-name NAME` | Query via a GSI/LSI instead of the base table. |
| `aws dynamodb batch-write-item --request-items file://F` | Write up to 25 items in one call (no conditions). |
| `aws dynamodb get-item --consistent-read` | Force a strongly consistent read (base table/LSI only). |
| `aws dynamodb update-table --billing-mode MODE` | Switch between on-demand and provisioned capacity. |
| `aws dynamodb update-table --stream-specification StreamEnabled=true,...` | Enable a change stream. |
| `aws dynamodb update-time-to-live` | Enable automatic item expiry. |

## Exercise

1. Create a table modeling orders with `customerId` (partition) +
   `orderId` (sort), plus a GSI on `orderStatus` + `createdAt`.
2. Insert at least 5 orders across 2 customers and a mix of statuses.
3. Query "all orders for one customer" against the base table, and "all
   orders in PLACED status, newest first" against the GSI — confirm each
   returns only the expected items.
4. Update one order's status with `update-item` and immediately re-query
   the GSI; note (in your own words) why the result is not guaranteed to
   reflect the update instantly.
5. Enable a stream with `NEW_AND_OLD_IMAGES`, then use
   `aws dynamodbstreams describe-stream` and
   `aws dynamodbstreams get-records` (via a shard iterator) to observe a
   raw change record after another `update-item`.
6. Enable TTL on an `expiresAt` attribute, insert one item with a past
   timestamp, and note it should disappear from `scan` results within the
   next day without you deleting it.
