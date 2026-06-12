# AWS EventBridge

**What it is:** A serverless event bus. Services publish events; **rules** match them by pattern and route to **targets** (Lambda, SQS, Step Functions, other buses).

**Why people use it:** Decouples producers from consumers via pub/sub — many consumers can react to one event without the producer knowing they exist. Content-based routing with no polling.

**Typically used for:** Event-driven architectures, fan-out (one event → N handlers), scheduled jobs (cron), reacting to AWS service events, and SaaS integrations.

> EventBridge vs SQS: **SQS** = point-to-point queue (one consumer group, you poll). **EventBridge** = pub/sub router (many targets, pattern-matched, pushed). Often used together: EventBridge rule → SQS queue → Lambda.

---

## Publish an event (SDK v3)

```bash
npm i @aws-sdk/client-eventbridge
```

```ts
import { EventBridgeClient, PutEventsCommand } from "@aws-sdk/client-eventbridge";

const eb = new EventBridgeClient({});

await eb.send(new PutEventsCommand({
  Entries: [{
    EventBusName: "app-bus",
    Source: "orders.service",          // your namespace
    DetailType: "OrderPlaced",         // the event name
    Detail: JSON.stringify({ orderId: "1", amount: 4200 }),
  }],
}));
```

An event has three routing keys: **`Source`**, **`DetailType`**, and the **`Detail`** payload (rules can match on any field).

---

## Rules — match & route

A rule's **event pattern** decides which events match; matched events go to its targets.

```json
{
  "source": ["orders.service"],
  "detail-type": ["OrderPlaced"],
  "detail": { "amount": [{ "numeric": [">", 1000] }] }
}
```

Only `OrderPlaced` events from `orders.service` with `amount > 1000` match. Non-matching events are silently dropped (no target, no cost).

---

## Consume in Lambda (typed)

```ts
import type { EventBridgeHandler } from "aws-lambda";

type OrderPlaced = { orderId: string; amount: number };

// EventBridgeHandler<DetailType, Detail, Return>
export const handler: EventBridgeHandler<"OrderPlaced", OrderPlaced, void> =
  async (event) => {
    const { orderId, amount } = event.detail;   // typed payload
    await fulfil(orderId, amount);
  };
```

Validate the detail with Zod via Powertools parser — the `EventBridgeSchema` envelope unwraps `detail` for you:

```ts
import { parser } from "@aws-lambda-powertools/parser/middleware";
import { EventBridgeEnvelope } from "@aws-lambda-powertools/parser/envelopes";
import { z } from "zod";

const Order = z.object({ orderId: z.string(), amount: z.number() });
export const handler = middy(async (order: z.infer<typeof Order>) => { /* ... */ })
  .use(parser({ schema: Order, envelope: EventBridgeEnvelope }));
```

> See [aws-lambda.md](aws-lambda.md) and [zod.md](zod.md).

---

## Advanced

### Terraform — bus, rule, Lambda target

```hcl
resource "aws_cloudwatch_event_bus" "app" {
  name = "app-bus"
}

resource "aws_cloudwatch_event_rule" "high_value_orders" {
  event_bus_name = aws_cloudwatch_event_bus.app.name
  event_pattern  = jsonencode({
    source        = ["orders.service"]
    "detail-type" = ["OrderPlaced"]
    detail        = { amount = [{ numeric = [">", 1000] }] }
  })
}

resource "aws_cloudwatch_event_target" "to_lambda" {
  rule           = aws_cloudwatch_event_rule.high_value_orders.name
  event_bus_name = aws_cloudwatch_event_bus.app.name
  arn            = aws_lambda_function.fulfil.arn
}

resource "aws_lambda_permission" "allow_eb" {
  statement_id  = "AllowEventBridge"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.fulfil.function_name
  principal     = "events.amazonaws.com"
  source_arn    = aws_cloudwatch_event_rule.high_value_orders.arn
}
```

> See [terraform.md](terraform.md).

### Scheduled rules (cron / rate)

EventBridge is the standard cron for serverless:

```hcl
resource "aws_cloudwatch_event_rule" "nightly" {
  schedule_expression = "cron(0 2 * * ? *)"    # 02:00 UTC daily — or rate(1 hour)
}
```

> For richer scheduling (timezones, one-off, flexible windows) use **EventBridge Scheduler**, the newer dedicated service.

### Fan-out (one event, many targets)

Attach multiple targets to one rule, or multiple rules to one event — every match fires independently. The core decoupling win: add a consumer without touching the producer.

```
OrderPlaced ──► rule ──► fulfilment Lambda
                    ├──► analytics SQS queue
                    └──► email Step Function
```

### Archive & replay

Archive matched events, then replay them onto the bus — to backfill a new consumer or recover from a bug.

```bash
aws events create-archive --archive-name orders --event-source-arn "$BUS_ARN"
aws events start-replay --replay-name fix-2024 --event-source-arn "$ARCHIVE_ARN" \
  --destination '{"Arn":"'$BUS_ARN'"}' \
  --event-start-time ... --event-end-time ...
```

### Reliability — add a DLQ to targets

Target delivery can fail; attach a DLQ so events aren't lost:

```hcl
resource "aws_cloudwatch_event_target" "to_lambda" {
  # ...
  dead_letter_config { arn = aws_sqs_queue.eb_dlq.arn }
  retry_policy { maximum_retry_attempts = 3 }
}
```

---

## Practical recipes

**Publish a test event (CLI):**
```bash
aws events put-events --entries '[{
  "Source":"orders.service","DetailType":"OrderPlaced",
  "Detail":"{\"orderId\":\"1\",\"amount\":4200}","EventBusName":"app-bus"
}]'
```

**List rules on a bus:**
```bash
aws events list-rules --event-bus-name app-bus
```

**Test an event pattern against a sample event (CLI):**
```bash
aws events test-event-pattern \
  --event-pattern '{"source":["orders.service"]}' \
  --event '{"source":"orders.service","detail-type":"OrderPlaced","detail":{}}'
```

**Send all events to CloudWatch Logs to debug what's flowing:**
```bash
# add a catch-all rule { "source": [{ "prefix": "" }] } targeting a log group
```

---

## Tips

- Namespace your `Source` (`orders.service`, `billing.service`) and use clear `DetailType` names — they're your routing contract.
- Match as narrowly as possible in the event pattern; do filtering in the rule, not in handler code.
- Attach a **DLQ + retry policy** to targets — delivery isn't guaranteed without one.
- EventBridge for pub/sub fan-out; put an SQS queue between a rule and Lambda when you need buffering/throttling.
- Use **EventBridge Scheduler** (not rules) for anything beyond simple cron — timezones, one-off schedules.
- Keep event payloads small and stable; treat `DetailType` + schema as a versioned contract consumers depend on.
