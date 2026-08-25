# Step Functions & Orchestration

Level 1 wired individual Lambda functions to triggers. Real workflows
chain many steps together — call function A, branch on its result, run
B and C in parallel, retry D if it fails, wait for a human approval.
**Step Functions** models this as a state machine, defined declaratively
in Amazon States Language (ASL, a JSON dialect).

## Why not just chain Lambdas from code?

You could have Lambda A invoke Lambda B directly, but then retry logic,
error handling, and the overall flow are buried in application code and
invisible without reading it. Step Functions makes the flow itself a
first-class, visualized resource, and it doesn't run any compute while
waiting (e.g., for `states:waitForTaskToken`) — you aren't billed for
Lambda sitting idle.

## Define a state machine

```json
{
  "Comment": "Order processing workflow",
  "StartAt": "ValidateOrder",
  "States": {
    "ValidateOrder": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:validateOrder",
      "Retry": [
        { "ErrorEquals": ["States.TaskFailed"], "IntervalSeconds": 2, "MaxAttempts": 3, "BackoffRate": 2.0 }
      ],
      "Next": "IsValid"
    },
    "IsValid": {
      "Type": "Choice",
      "Choices": [
        { "Variable": "$.valid", "BooleanEquals": true, "Next": "ProcessPayment" }
      ],
      "Default": "RejectOrder"
    },
    "ProcessPayment": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:chargeCard",
      "Catch": [
        { "ErrorEquals": ["States.ALL"], "Next": "RejectOrder" }
      ],
      "Next": "ShipOrder"
    },
    "ShipOrder": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:shipOrder",
      "End": true
    },
    "RejectOrder": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:rejectOrder",
      "End": true
    }
  }
}
```

`Retry` and `Catch` are attached per-state — you get exponential backoff
and error branching without writing any try/except code.

## Create and run it

```bash
aws stepfunctions create-state-machine \
  --name order-processing \
  --definition file://state-machine.json \
  --role-arn arn:aws:iam::123456789012:role/StepFunctionsExecutionRole \
  --type STANDARD

aws stepfunctions start-execution \
  --state-machine-arn arn:aws:states:us-east-1:123456789012:stateMachine:order-processing \
  --input '{"orderId": "ord-8842", "amount": 49.99}'
# {
#     "executionArn": "arn:aws:states:us-east-1:123456789012:execution:order-processing:b1e2c3d4",
#     "startDate": 1732000000.123
# }

aws stepfunctions describe-execution \
  --execution-arn arn:aws:states:us-east-1:123456789012:execution:order-processing:b1e2c3d4 \
  --query 'status'
# "SUCCEEDED"
```

## Standard vs. Express workflows

| | Standard | Express |
|---|---|---|
| Max duration | 1 year | 5 minutes |
| Execution history | Retained, viewable per-run | CloudWatch Logs only |
| Pricing | Per state transition | Per invocation + duration |
| Semantics | Exactly-once | At-least-once |
| Use case | Long orchestrations, human approval steps | High-volume, short event processing |

`--type EXPRESS` swaps the execution model entirely — pick Standard for
anything with `Wait` states measured in hours/days or where you need to
inspect individual past executions in the console; Express for
high-throughput, short-lived event pipelines where per-transition
billing would be too expensive.

## Parallel and Map states

```json
"NotifyAll": {
  "Type": "Parallel",
  "Branches": [
    { "StartAt": "Email", "States": { "Email": { "Type": "Task", "Resource": "...", "End": true } } },
    { "StartAt": "SMS", "States": { "SMS": { "Type": "Task", "Resource": "...", "End": true } } }
  ],
  "Next": "Done"
}
```

A `Map` state runs the same sub-workflow over every item in an array
(e.g., process each line item in an order) — with `MaxConcurrency` to
cap parallelism and avoid overwhelming downstream systems or Lambda
concurrency limits.

## Gotchas

- **The execution role needs `lambda:InvokeFunction` on every Task
  resource it calls** — a common failure mode is a state machine that
  starts fine but fails on the second Lambda because only the first
  ARN was granted.
- **Input/output is JSON passed between states** — a Lambda that
  returns a non-JSON-serializable value, or one whose output doesn't
  match what the next state's `InputPath`/`Parameters` expects, fails
  silently into a generic `States.Runtime` error.
- **Standard workflow state transitions are billed individually** — a
  `Map` state iterating over 10,000 items can rack up costs fast;
  Express or batching is usually cheaper for high-fan-out cases.
- **`Wait` states with `SecondsPath`/`Timestamp` still count toward the
  1-year Standard execution limit** — long-running approval workflows
  need a plan for what happens if nobody approves in time.
- Editing a state machine's definition does **not** affect executions
  already in progress — they keep running against the definition that
  was active when they started.

## Cheat sheet

| Command | Purpose |
|---|---|
| `aws stepfunctions create-state-machine` | Deploy a new definition |
| `aws stepfunctions update-state-machine` | Update an existing one |
| `aws stepfunctions start-execution` | Trigger a run |
| `aws stepfunctions describe-execution` | Check status/output |
| `aws stepfunctions get-execution-history` | Full step-by-step trace |
| `aws stepfunctions stop-execution` | Cancel a running execution |

## Exercise

Write an ASL definition with a `Choice` state that routes an input
`{"score": N}` to one of three Lambda ARNs depending on whether `score`
is below 50, between 50-80, or above 80. Deploy it as a Standard
workflow and run three executions to hit all three branches, verifying
each via `describe-execution --query output`.
