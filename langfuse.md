# Langfuse

**What it is:** An open-source observability and evaluation platform for LLM applications.

**Why people use it:** It records prompts, model calls, tool calls, costs, latency, errors, scores, and user/session metadata so LLM behaviour is debuggable.

**Typically used for:** Tracing AI apps, inspecting agent runs, monitoring production quality, prompt management, evaluations, datasets, and cost/latency analysis.

> This sheet focuses on Langfuse JS/TS. Source: https://langfuse.com/docs

---

## Mental model

A **trace** is one end-to-end run (an API request, an agent invocation). It contains nested **observations**:

- **spans** — units of work (a retrieval step, a tool call)
- **generations** — LLM calls (model, tokens, cost, latency)

…forming a tree you inspect step by step. **Scores** attach to a trace or observation for eval/quality. Tracing is OpenTelemetry-based, so nested calls become nested spans automatically.

---

## Setup

```bash
npm i @langfuse/tracing @langfuse/otel @opentelemetry/sdk-node
```

```bash
export LANGFUSE_SECRET_KEY=sk-lf-...
export LANGFUSE_PUBLIC_KEY=pk-lf-...
export LANGFUSE_BASE_URL=https://cloud.langfuse.com
```

For US or self-hosted regions, set the matching base URL.

> Langfuse's current docs list JS/TS SDK v5 and OpenTelemetry-based packages: `@langfuse/tracing`, `@langfuse/otel`, and `@opentelemetry/sdk-node`.

---

## Most common packages

```bash
npm i @langfuse/tracing @langfuse/otel @opentelemetry/sdk-node
npm i @langfuse/openai             # auto-trace OpenAI JS SDK calls
npm i @langfuse/langchain          # LangChain/LangGraph callback handler
npm i @langfuse/client             # prompts, datasets, scores
```

## Initialise OpenTelemetry

```ts
// instrumentation.ts
import { LangfuseSpanProcessor } from "@langfuse/otel";
import { NodeSDK } from "@opentelemetry/sdk-node";

export const sdk = new NodeSDK({
  spanProcessors: [new LangfuseSpanProcessor()],
});

sdk.start();
```

Import this before app code:

```ts
import "./instrumentation.js";
```

Flush on shutdown:

```ts
process.on("SIGTERM", async () => {
  await sdk.shutdown();
  process.exit(0);
});
```

## Manual trace/span

```ts
import { startActiveObservation } from "@langfuse/tracing";

await startActiveObservation("answer-question", async (span) => {
  span.update({
    input: { question: "What is RAG?" },
  });

  const answer = "Retrieval augmented generation.";

  span.update({
    output: { answer },
    metadata: { route: "/api/answer" },
  });
});
```

## OpenAI auto-tracing

```bash
npm i @langfuse/openai @opentelemetry/sdk-node
```

```ts
import "./instrumentation.js";
import OpenAI from "openai";
import { observeOpenAI } from "@langfuse/openai";

const openai = observeOpenAI(new OpenAI());

const res = await openai.chat.completions.create({
  model: "gpt-4o-mini",
  messages: [{ role: "user", content: "Say hello" }],
});
```

> Langfuse documents `@langfuse/openai` as a JS/TS wrapper around the official OpenAI SDK.

## LangChain / LangGraph tracing

```bash
npm i @langfuse/core @langfuse/langchain
```

```ts
import { CallbackHandler } from "@langfuse/langchain";

const langfuseHandler = new CallbackHandler();

await chain.invoke(input, {
  callbacks: [langfuseHandler],
});
```

For LangGraph:

```ts
await graph.invoke(input, {
  callbacks: [langfuseHandler],
  metadata: { userId: "u_123" },
});
```

> Langfuse's LangChain integration uses callbacks to capture chains, LLMs, tools, retrievers, and LangGraph runs.

---

## Common trace attributes

```ts
span.update({
  name: "support-answer",
  input: { message: "Refund?" },
  output: { answer: "..." },
  metadata: {
    route: "/api/support",
    model: "gpt-4o-mini",
  },
});
```

Use consistent IDs:

```ts
span.update({
  userId: "user_123",
  sessionId: "chat_456",
  tags: ["prod", "support"],
});
```

## Mask sensitive data

Filter before sending, not after:

```ts
const redact = (value: string) =>
  value.replace(/\b\d{4}[- ]?\d{4}[- ]?\d{4}[- ]?\d{4}\b/g, "[card]");

span.update({
  input: { message: redact(userMessage) },
});
```

## Scores

Use scores for quality, thumbs-up/down, eval results, or policy checks.

```ts
import { LangfuseClient } from "@langfuse/client";

const langfuse = new LangfuseClient();

await langfuse.score.create({
  traceId: "trace-id",
  name: "user-feedback",
  value: 1,
  comment: "Helpful answer",
});
```

## Prompt management

```ts
import { LangfuseClient } from "@langfuse/client";

const langfuse = new LangfuseClient();
const prompt = await langfuse.prompt.get("support-answer");

const compiled = prompt.compile({
  product: "Billing",
  question: "How do refunds work?",
});
```

---

## Python quick start

```bash
pip install langfuse
```

```python
from langfuse import get_client

langfuse = get_client()

with langfuse.start_as_current_observation(name="answer-question") as span:
    span.update(input={"question": "What is RAG?"})
    span.update(output={"answer": "Retrieval augmented generation."})

langfuse.flush()
```

OpenAI Python drop-in:

```python
from langfuse.openai import OpenAI

client = OpenAI()

client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[{"role": "user", "content": "Say hello"}],
)
```

---

## Practical recipes

**Add tracing to a Node service:**
```ts
// index.ts
import "./instrumentation.js";
import { app } from "./app.js";
```

**Flush traces in a short-lived script:**
```ts
await main();
await sdk.shutdown();
```

**Tag production traces:**
```ts
span.update({ tags: ["prod", "api"] });
```

**Attach user/session metadata:**
```ts
span.update({
  userId: user.id,
  sessionId: conversation.id,
});
```

**Trace one LangChain call:**
```ts
await chain.invoke(input, { callbacks: [new CallbackHandler()] });
```

**Trace one OpenAI client with minimal app changes:**
```ts
const openai = observeOpenAI(new OpenAI());
```

**Disable noisy dev tracing locally:**
```bash
unset LANGFUSE_PUBLIC_KEY LANGFUSE_SECRET_KEY
```

---

## Tips

- Initialise instrumentation before importing code that creates model clients.
- Always flush/shutdown in CLIs, tests, and short-lived workers.
- Use `userId`, `sessionId`, tags, and metadata consistently; they make traces searchable.
- Redact secrets and PII before sending inputs/outputs.
- Trace retrieval and tool calls, not just final model calls.
- Keep Langfuse optional in local dev so missing credentials do not block normal work.
