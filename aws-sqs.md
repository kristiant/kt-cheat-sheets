# AWS SQS

**What it is:** A fully managed message queue. Producers send messages; consumers poll and process them; SQS handles storage, delivery, and retries.

**Why people use it:** Decouples producers from consumers, absorbs traffic spikes (buffer), and gives reliable at-least-once delivery with automatic retries and dead-letter queues.

**Typically used for:** Async job processing, smoothing load between services, Lambda fan-out, and decoupling slow work (emails, parsing, embeddings) from request paths.

> Standard vs FIFO: **Standard** = high throughput, at-least-once, best-effort ordering. **FIFO** = exactly-once processing, strict ordering, lower throughput. Default to Standard unless you genuinely need ordering.

---

## Send & receive (SDK v3)

```bash
npm i @aws-sdk/client-sqs
```

```ts
import { SQSClient, SendMessageCommand, ReceiveMessageCommand, DeleteMessageCommand }
  from "@aws-sdk/client-sqs";

const sqs = new SQSClient({});
const QueueUrl = process.env.QUEUE_URL!;

// produce
await sqs.send(new SendMessageCommand({
  QueueUrl,
  MessageBody: JSON.stringify({ orderId: "1" }),
}));

// consume (manual polling — usually you let Lambda do this instead)
const { Messages } = await sqs.send(new ReceiveMessageCommand({
  QueueUrl, MaxNumberOfMessages: 10, WaitTimeSeconds: 20,   // long polling
}));
for (const m of Messages ?? []) {
  await handle(JSON.parse(m.Body!));
  await sqs.send(new DeleteMessageCommand({ QueueUrl, ReceiptHandle: m.ReceiptHandle! }));
}
```

> A message stays invisible (`VisibilityTimeout`) while being processed; if you don't delete it in time, it reappears for retry. Set the timeout > your processing time.

---

## Lambda consumer + partial batch failures

The standard pattern: SQS triggers Lambda with a batch. If one message fails, you don't want the **whole batch** retried — only the failures. Powertools' batch utility + `ReportBatchItemFailures` does this cleanly.

```bash
npm i @aws-lambda-powertools/batch
```

```ts
import {
  BatchProcessor, EventType, processPartialResponse,
} from "@aws-lambda-powertools/batch";
import type { SQSRecord, SQSHandler } from "aws-lambda";

const processor = new BatchProcessor(EventType.SQS);

const recordHandler = async (record: SQSRecord) => {
  const body = JSON.parse(record.body);
  await doWork(body);            // throw to mark THIS record failed
};

export const handler: SQSHandler = async (event, context) =>
  processPartialResponse(event, recordHandler, processor, { context });
```

This returns `{ batchItemFailures: [...] }` — only the failed message IDs go back to the queue. Requires the event-source mapping to have **`ReportBatchItemFailures` enabled** (see Terraform below).

> **Grounded:** This is AWS's [recommended partial-batch-response pattern](https://docs.aws.amazon.com/prescriptive-guidance/latest/lambda-event-filtering-partial-batch-responses-for-sqs/). Without it, one poison message forces the entire batch to redrive. See [aws-lambda.md](aws-lambda.md).

For FIFO queues use `SqsFifoPartialProcessor` (stops at the first failure to preserve order).

---

## Advanced

### Dead-letter queue (DLQ)

After `maxReceiveCount` failed attempts, SQS moves a message to the DLQ instead of retrying forever — so poison messages don't block the queue. **Always configure one.**

### Terraform — queue + DLQ + Lambda trigger

```hcl
resource "aws_sqs_queue" "dlq" {
  name = "ingest-dlq"
}

resource "aws_sqs_queue" "ingest" {
  name                       = "ingest"
  visibility_timeout_seconds = 180          # > Lambda timeout
  redrive_policy = jsonencode({
    deadLetterTargetArn = aws_sqs_queue.dlq.arn
    maxReceiveCount     = 5                  # → DLQ after 5 failed attempts
  })
}

resource "aws_lambda_event_source_mapping" "trigger" {
  event_source_arn                   = aws_sqs_queue.ingest.arn
  function_name                      = aws_lambda_function.worker.arn
  batch_size                         = 10
  function_response_types            = ["ReportBatchItemFailures"]   # enables partial failures
  maximum_batching_window_in_seconds = 5
}
```

> See [terraform.md](terraform.md).

### Batch send (up to 10 per call, cheaper)

```ts
import { SendMessageBatchCommand } from "@aws-sdk/client-sqs";

await sqs.send(new SendMessageBatchCommand({
  QueueUrl,
  Entries: orders.map((o, i) => ({ Id: String(i), MessageBody: JSON.stringify(o) })),
}));
```

### FIFO specifics

```ts
await sqs.send(new SendMessageCommand({
  QueueUrl,                                  // must end in .fifo
  MessageBody: JSON.stringify(order),
  MessageGroupId: order.customerId,          // ordering scope — parallel across groups
  MessageDeduplicationId: order.id,          // dedup window (5 min)
}));
```

### Delay & long polling

```ts
// per-message delay (Standard queues)
new SendMessageCommand({ QueueUrl, MessageBody, DelaySeconds: 30 });
// long polling — fewer empty receives, lower cost
new ReceiveMessageCommand({ QueueUrl, WaitTimeSeconds: 20 });
```

---

## Practical recipes

**Send a test message (CLI):**
```bash
aws sqs send-message --queue-url "$Q" --message-body '{"orderId":"1"}'
```

**Peek at messages without deleting (CLI):**
```bash
aws sqs receive-message --queue-url "$Q" --max-number-of-messages 10 --wait-time-seconds 5
```

**Check queue depth (backlog + in-flight):**
```bash
aws sqs get-queue-attributes --queue-url "$Q" \
  --attribute-names ApproximateNumberOfMessages ApproximateNumberOfMessagesNotVisible
```

**Redrive a DLQ back to the source (CLI):**
```bash
aws sqs start-message-move-task \
  --source-arn "$DLQ_ARN" --destination-arn "$SRC_ARN"
```

**Purge a queue (⚠️ deletes all messages):**
```bash
aws sqs purge-queue --queue-url "$Q"
```

---

## Tips

- Always pair a queue with a **DLQ** + `maxReceiveCount`; otherwise poison messages retry forever.
- `VisibilityTimeout` must exceed your processing time, or messages reappear mid-process and get processed twice.
- Use `ReportBatchItemFailures` + Powertools batch utility so one bad message doesn't redrive the whole batch.
- Make consumers **idempotent** — at-least-once delivery means duplicates happen (see Lambda idempotency).
- Long polling (`WaitTimeSeconds: 20`) over short polling — fewer empty calls, lower cost.
- FIFO only when ordering/exactly-once is a real requirement; it caps throughput.
