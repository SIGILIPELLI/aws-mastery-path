# Advanced Monitoring (X-Ray & Tracing)

Level 1's CloudWatch gave you metrics and logs per resource. Once a
request touches five services — API Gateway, Lambda, DynamoDB, SQS,
another Lambda — a slow request could be stuck anywhere in that chain,
and logs alone don't show you the chain. **AWS X-Ray** provides
distributed tracing: one trace ID follows a request across every
service it touches.

## How tracing works

Each service call annotated with X-Ray emits a **segment** (or
**subsegment** for calls within a service, like an outbound HTTP call
or DB query). X-Ray stitches segments sharing a trace ID into a single
**trace**, visualized as a **service map**.

## Enable X-Ray on Lambda

```bash
aws lambda update-function-configuration \
  --function-name validateOrder \
  --tracing-config Mode=Active
```

`Mode=Active` samples and traces every invocation (subject to X-Ray's
default sampling rule); `PassThrough` only traces if the incoming
request already carries a trace header, useful when an upstream
service (like API Gateway) is doing the sampling decision.

Inside the function, instrument outbound calls with the X-Ray SDK:

```python
from aws_xray_sdk.core import xray_recorder, patch_all
patch_all()  # auto-instruments boto3, requests, sqlite3, etc.

def handler(event, context):
    with xray_recorder.capture("validate_business_rules"):
        result = run_validation(event)
    return result
```

`patch_all()` wraps boto3 calls automatically — a DynamoDB `get_item`
inside the handler shows up as its own subsegment with latency, with no
manual instrumentation needed for that part.

## Enable tracing on API Gateway

```bash
aws apigateway update-stage \
  --rest-api-id abc123def4 \
  --stage-name prod \
  --patch-operations op=replace,path=/tracingEnabled,value=true
```

## Query traces

```bash
aws xray get-trace-summaries \
  --start-time $(date -u -d '1 hour ago' +%s) \
  --end-time $(date -u +%s) \
  --filter-expression 'responsetime > 3'
# {
#     "TraceSummaries": [
#         { "Id": "1-66c3a1f2-abcdef1234567890abcdef12", "Duration": 4.21, "ResponseTime": 4.21, "HasError": false }
#     ]
# }

aws xray batch-get-traces --trace-ids 1-66c3a1f2-abcdef1234567890abcdef12
```

The `filter-expression` syntax lets you query by response time, HTTP
status, annotation key/value, or service name — `filter-expression
'service("validateOrder") { fault }'` finds only traces where that
specific service raised an error.

## Annotations vs. metadata

```python
xray_recorder.current_segment().put_annotation("customer_tier", "enterprise")
xray_recorder.current_segment().put_metadata("request_payload", event)
```

**Annotations** are indexed and filterable in trace queries (small
key-value pairs only — strings, numbers, booleans). **Metadata** is
stored but not indexed or searchable — use it for larger debugging
payloads you want visible on a trace but don't need to query by.

## Gotchas

- **Sampling means not every request is traced** — the default rule
  traces the first request per second plus 5% of additional requests;
  under real load you will *not* see 100% of invocations in X-Ray
  unless you configure a custom sampling rule with a higher rate (which
  increases cost).
- **X-Ray tracing needs IAM permission** (`xray:PutTraceSegments`,
  `xray:PutTelemetryRecords`) on the Lambda execution role — enabling
  `Mode=Active` without it silently drops all trace data.
- **Cold starts show up as latency in the trace but are a separate
  metric** (`Init Duration` in Lambda's own logs) — X-Ray's segment
  duration includes cold start time unless you specifically look at
  the `Initialization` subsegment.
- **Traces are billed per trace recorded and per trace retrieved** —
  enabling `Active` mode on every function in a high-traffic system
  without tuning sampling can become a meaningful line item.
- **Cross-account/cross-service traces require the trace header to
  propagate** — if a service calls another via a mechanism that
  strips HTTP headers (e.g., certain SQS message attributes setups),
  the trace chain breaks into two disconnected traces.

## Cheat sheet

| Command | Purpose |
|---|---|
| `aws lambda update-function-configuration --tracing-config` | Enable tracing on Lambda |
| `aws apigateway update-stage ... tracingEnabled` | Enable tracing on API Gateway |
| `aws xray get-trace-summaries` | Query recent traces |
| `aws xray batch-get-traces` | Fetch full trace detail |
| `aws xray get-service-graph` | Fetch the service map data |
| `aws xray put-trace-segments` | Manually submit segments (custom instrumentation) |

## How It Actually Works

X-Ray reconstructs a distributed request's full call graph from
independently-emitted **trace segments** — each service involved in
handling a request (via the X-Ray SDK or auto-instrumentation) generates its
own segment describing the work it did, tagged with a shared trace ID that's
propagated forward through request headers (`X-Amzn-Trace-Id`) as the
request hops between services. X-Ray's backend never sees the request
travel end-to-end itself; it only receives these disconnected segments
(often via a local X-Ray daemon that batches and forwards them
asynchronously, specifically so tracing overhead doesn't block your
application's actual response path) and reassembles them into one trace
purely by matching that shared trace ID — which is exactly why a broken
trace (a service that fails to propagate the header) produces a visibly
truncated trace map rather than an error: X-Ray has no way to know a hop it
was never told about exists.

**Sampling** exists because instrumenting and transmitting a segment for
every single request at scale would itself become a meaningful source of
load and cost; the default X-Ray sampling rule (1 request/second plus 5% of
additional requests) is evaluated locally by the SDK per-request against a
sampling decision cached from the X-Ray control plane, meaning the decision
to trace a given request is made *before* the request executes, not
after-the-fact based on whether something interesting happened to it —
which is why intermittent errors can be invisible in X-Ray traces unless you
tune sampling rules or force-sample on error conditions explicitly.

CloudWatch's **Container Insights / embedded metric format** takes a
different approach entirely: rather than a separate tracing pipeline,
structured JSON log lines emitted by your application are parsed by
CloudWatch Logs at ingestion time, with fields you mark as metrics
extracted and written directly into the CloudWatch metrics store — meaning
"custom metrics" from EMF logs are derived from log data after the fact by
CloudWatch's own ingestion pipeline, not sent as a distinct metric API call
from your code.

## Exercise

Enable `Active` X-Ray tracing on a Lambda function fronted by API
Gateway, instrument one outbound call (e.g., to DynamoDB) with
`patch_all()`, add a `customer_tier` annotation, then use
`get-trace-summaries` with a `filter-expression` to find only traces
where `customer_tier = "enterprise"` and response time exceeded 1
second.
