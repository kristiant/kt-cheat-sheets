# AWS Bedrock

**What it is:** A managed service for calling foundation models (Anthropic Claude, Amazon Nova, Meta Llama, Cohere, etc.) through one AWS API — no model hosting.

**Why people use it:** Access many models behind one API with AWS-native auth (IAM), data residency, and no data used for training. Switch models by changing an ID.

**Typically used for:** Chat/RAG/agent backends that need enterprise controls, multi-model flexibility, or to keep LLM traffic inside AWS.

> The **Converse API** is the modern, unified interface — same request shape across all models, with built-in tool use and streaming. Prefer it over the older per-model `InvokeModel`.

---

## Converse — one call (SDK v3)

```bash
npm i @aws-sdk/client-bedrock-runtime
```

```ts
import { BedrockRuntimeClient, ConverseCommand } from "@aws-sdk/client-bedrock-runtime";

const client = new BedrockRuntimeClient({ region: "us-east-1" });

const res = await client.send(new ConverseCommand({
  modelId: "anthropic.claude-3-5-sonnet-20240620-v1:0",
  messages: [{ role: "user", content: [{ text: "Summarise RAG in one sentence." }] }],
  inferenceConfig: { maxTokens: 512, temperature: 0 },
}));

console.log(res.output?.message?.content?.[0]?.text);
```

Same shape works for Nova, Llama, Cohere — just change `modelId`.

---

## Streaming

```ts
import { ConverseStreamCommand } from "@aws-sdk/client-bedrock-runtime";

const res = await client.send(new ConverseStreamCommand({
  modelId,
  messages: [{ role: "user", content: [{ text: "Write a haiku." }] }],
}));

for await (const event of res.stream ?? []) {
  const text = event.contentBlockDelta?.delta?.text;
  if (text) process.stdout.write(text);
}
```

---

## Via LangChain

If you're already on LangChain, `ChatBedrockConverse` wraps the Converse API with the standard chat interface — drops into chains/graphs unchanged.

```bash
npm i @langchain/aws
```

```ts
import { ChatBedrockConverse } from "@langchain/aws";

const llm = new ChatBedrockConverse({
  model: "anthropic.claude-3-5-sonnet-20240620-v1:0",
  region: "us-east-1",
  temperature: 0,
});

const res = await llm.invoke("Explain idempotency in one line.");
```

> Use this in the [agentic-patterns.md](agentic-patterns.md) / [rag.md](rag.md) examples to swap a hosted model for a Bedrock-backed one — the rest of the graph is identical.

---

## Advanced

### Tool use (function calling)

Converse has native tool calling. Declare tools, the model returns a `toolUse` block, you run it and feed back a `toolResult`.

```ts
const res = await client.send(new ConverseCommand({
  modelId,
  messages,
  toolConfig: {
    tools: [{
      toolSpec: {
        name: "get_weather",
        description: "Current weather for a city",
        inputSchema: { json: {
          type: "object",
          properties: { city: { type: "string" } },
          required: ["city"],
        } },
      },
    }],
  },
}));

if (res.stopReason === "tool_use") {
  const call = res.output?.message?.content?.find((c) => c.toolUse)?.toolUse;
  // run call.name with call.input, then send a toolResult message back
}
```

### System prompt & multi-turn

```ts
new ConverseCommand({
  modelId,
  system: [{ text: "You are a terse assistant. Answer in <20 words." }],
  messages: [
    { role: "user", content: [{ text: "What is SQS?" }] },
    { role: "assistant", content: [{ text: "A managed message queue." }] },
    { role: "user", content: [{ text: "And EventBridge?" }] },   // multi-turn context
  ],
});
```

### Bedrock Knowledge Bases (managed RAG)

Bedrock can host the whole RAG pipeline — ingest to a vector store and retrieve in one call (`RetrieveAndGenerate`) instead of wiring chunking/embedding/retrieval yourself. Good when you want managed RAG; roll your own (see [rag.md](rag.md)) when you need control over retrieval.

### IAM — least privilege

```json
{
  "Effect": "Allow",
  "Action": ["bedrock:InvokeModel", "bedrock:InvokeModelWithResponseStream"],
  "Resource": "arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-3-5-sonnet-*"
}
```

Models must be **enabled** in the Bedrock console (model access) per account/region before use.

---

## Practical recipes

**List models available in a region (CLI):**
```bash
aws bedrock list-foundation-models --region us-east-1 \
  --query "modelSummaries[].modelId" --output text
```

**One-shot prompt from the CLI:**
```bash
aws bedrock-runtime converse --model-id "$MODEL" \
  --messages '[{"role":"user","content":[{"text":"hello"}]}]'
```

**Call a Bedrock model inside a Lambda (reuse the client):**
```ts
const client = new BedrockRuntimeClient({});   // module scope — see aws-lambda.md
export const handler = async (event) =>
  client.send(new ConverseCommand({ modelId, messages: toMessages(event) }));
```

**Force JSON output (prompt + parse):**
```ts
system: [{ text: "Respond ONLY with valid JSON matching {answer: string}." }]
// then JSON.parse the text block — validate with Zod (see zod.md)
```

---

## Tips

- Use the **Converse API**, not `InvokeModel` — uniform shape across models, native tool use, easy model swaps.
- Enable model access in the console first; calls fail with AccessDenied until you do.
- Model availability and IDs are **region-specific** — check `list-foundation-models` per region.
- Reuse the `BedrockRuntimeClient` at module scope in Lambda; don't construct per request.
- Scope IAM to specific model ARNs, not `bedrock:* / *`.
- On LangChain? `ChatBedrockConverse` swaps in without touching your chains/graphs.
- Validate model output (Zod) before trusting it — structure isn't guaranteed even with a JSON instruction.
