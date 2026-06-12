# AWS Lambda

**What it is:** A serverless compute service that runs your function code in response to events, with no servers to manage and pay-per-invocation billing.

**Why people use it:** No infrastructure to run or scale, scales to zero (and to thousands concurrently), and integrates natively with the rest of AWS (SQS, EventBridge, API Gateway, S3).

**Typically used for:** Event handlers (queue/stream consumers), API backends, scheduled jobs, glue between AWS services, and async pipeline stages.

> TS-first. Examples use [Powertools for AWS Lambda](https://docs.powertools.aws.dev/lambda/typescript/latest/) (observability + best-practice utilities), [middy](https://middy.js.org) (middleware), and [http-errors](https://github.com/jshttp/http-errors).

---

## The handler

```ts
import type { Handler } from "aws-lambda";

export const handler: Handler = async (event, context) => {
  return { statusCode: 200, body: JSON.stringify({ ok: true }) };
};
```

Typed events — use the specific event type for the trigger:

```ts
import type { SQSHandler, APIGatewayProxyHandlerV2, EventBridgeHandler } from "aws-lambda";

export const fromQueue: SQSHandler = async (event) => { /* event.Records */ };
export const fromApi: APIGatewayProxyHandlerV2 = async (event) => { /* event.body */ };
```

---

## Powertools — observability baseline

The three you always want. `injectLambdaContext` adds request IDs to every log; metrics/traces flush automatically.

```bash
npm i @aws-lambda-powertools/logger @aws-lambda-powertools/tracer @aws-lambda-powertools/metrics @middy/core
```

```ts
import middy from "@middy/core";
import { Logger, injectLambdaContext } from "@aws-lambda-powertools/logger";
import { Tracer, captureLambdaHandler } from "@aws-lambda-powertools/tracer/middleware";
import { Metrics, MetricUnit, logMetrics } from "@aws-lambda-powertools/metrics";

const logger = new Logger({ serviceName: "orders" });
const tracer = new Tracer({ serviceName: "orders" });
const metrics = new Metrics({ namespace: "shop", serviceName: "orders" });

export const handler = middy(async (event) => {
  logger.info("processing", { event });
  metrics.addMetric("OrdersProcessed", MetricUnit.Count, 1);
  return { ok: true };
})
  .use(captureLambdaHandler(tracer))      // tracer first — wraps everything in a segment
  .use(injectLambdaContext(logger, { logEvent: true }))
  .use(logMetrics(metrics));
```

> Middleware order matters: **tracer → logger → others**. The tracer opens the X-Ray segment; the logger wants to log the triggering event inside it.

---

## HTTP errors (API Gateway handlers)

Throw semantic errors with `http-errors`; let middy's error handler turn them into proper HTTP responses — no manual `statusCode` plumbing.

```bash
npm i http-errors @middy/http-error-handler @middy/http-json-body-parser
```

```ts
import createError from "http-errors";
import httpErrorHandler from "@middy/http-error-handler";
import jsonBodyParser from "@middy/http-json-body-parser";

export const handler = middy(async (event) => {
  const { id } = event.pathParameters ?? {};
  const order = await db.find(id);
  if (!order) throw new createError.NotFound(`Order ${id} not found`);   // → 404
  return { statusCode: 200, body: JSON.stringify(order) };
})
  .use(jsonBodyParser())        // parses + throws 422 on bad JSON
  .use(httpErrorHandler());     // maps thrown HttpErrors → status + body
```

Common throws: `BadRequest` (400), `Unauthorized` (401), `Forbidden` (403), `NotFound` (404), `Conflict` (409), `UnprocessableEntity` (422).

---

## Advanced

### Idempotency (exactly-once on retries)

Lambda retries on failure and SQS can deliver twice — make handlers safe to re-run. Powertools stores results in DynamoDB keyed off the payload.

```bash
npm i @aws-lambda-powertools/idempotency
```

```ts
import { makeIdempotent } from "@aws-lambda-powertools/idempotency";
import { DynamoDBPersistenceLayer } from "@aws-lambda-powertools/idempotency/dynamodb";

const persistence = new DynamoDBPersistenceLayer({ tableName: "idempotency" });

export const handler = makeIdempotent(
  async (event) => chargeCard(event),    // runs once per unique payload; cached result on retry
  { persistenceStore: persistence },
);
```

### Input validation with Zod (parser)

Parse + validate the event at the boundary; reject malformed input early.

```ts
import { parser } from "@aws-lambda-powertools/parser/middleware";
import { z } from "zod";

const Schema = z.object({ orderId: z.string(), amount: z.number().positive() });

export const handler = middy(async (event: z.infer<typeof Schema>) => { /* typed + valid */ })
  .use(parser({ schema: Schema }));
```

> See [zod.md](zod.md). Single schema = runtime validation + the handler's static type.

### Fetch secrets / params (cached)

Powertools caches parameter/secret fetches so you're not hitting the API every invocation.

```ts
import { getSecret } from "@aws-lambda-powertools/parameters/secrets";

const { apiKey } = JSON.parse(await getSecret("prod/openai", { maxAge: 300 }));   // 5-min cache
```

> See [aws-secrets-manager.md](aws-secrets-manager.md).

### Cold starts

- Keep the bundle small (tree-shake, esbuild) — less code to load.
- Init SDK clients/DB pools **outside** the handler so they're reused across warm invocations.
- Reach for provisioned concurrency only when p99 latency on cold starts actually matters.

```ts
const sqs = new SQSClient({});   // module scope — created once, reused
export const handler = async () => { await sqs.send(...); };
```

---

## Practical recipes

**Invoke a function locally / in the cloud (CLI):**
```bash
aws lambda invoke --function-name orders --payload '{"orderId":"1"}' out.json
```

**Tail logs live:**
```bash
aws logs tail /aws/lambda/orders --follow
```

**Update function code from a zip:**
```bash
aws lambda update-function-code --function-name orders --zip-file fileb://dist.zip
```

**Set env vars without redeploying code:**
```bash
aws lambda update-function-configuration --function-name orders \
  --environment "Variables={LOG_LEVEL=DEBUG,STAGE=prod}"
```

**Reserve concurrency (protect a downstream DB from overload):**
```bash
aws lambda put-function-concurrency --function-name orders --reserved-concurrent-executions 10
```

---

## Tips

- Init clients/pools at module scope; only per-request work belongs in the handler.
- Make handlers idempotent — SQS at-least-once delivery and Lambda retries mean double-invocation happens.
- Validate input at the edge (parser + Zod); never trust the event shape.
- Set a **dead-letter queue** or `onFailure` destination so failed async events aren't lost silently.
- Use Powertools logger over `console.log` — structured JSON logs are queryable in CloudWatch.
- Keep functions single-purpose; one trigger, one job.
- Set a realistic `timeout` and `memorySize` — more memory also means more CPU.
