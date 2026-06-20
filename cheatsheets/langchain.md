# LangChain

**What it is:** A JavaScript/TypeScript framework for building LLM apps with models, prompts, tools, retrievers, and agents.

**Why people use it:** It gives standard building blocks for wiring LLMs to real data and actions without rewriting the same glue code.

**Typically used for:** RAG pipelines, structured extraction, chatbots, tool calling, model-provider switching, and as the building blocks under a LangGraph agent.

> **LangChain is the building blocks; LangGraph is the orchestration.** LangChain (JS/TS) gives you the components — models, prompts, parsers, retrievers, tools, runnables — for *single-flow* execution (`input → step → step → output`). Once you need loops, branching, persistent state, or multi-agent flow, you move to [LangGraph](langgraph.md). Most examples from 2024 on end up there.
> Source: https://docs.langchain.com/oss/javascript/langchain/overview

---

## Setup

```bash
npm i langchain @langchain/core zod
npm i @langchain/openai              # OpenAI integration
```

```bash
export OPENAI_API_KEY=sk-...
```

```ts
import { ChatOpenAI } from "@langchain/openai";

const model = new ChatOpenAI({
  model: "gpt-4o-mini",
  temperature: 0,
});
```

---

**Packages:** `@langchain/openai` `@langchain/anthropic` (model providers) · `@langchain/community` (integrations) · `@langchain/textsplitters` · `@langchain/langgraph`.

---

## Runnables

Models, prompts, parsers, retrievers, your own functions, and entire chains are all **Runnables** — they share one interface, so they compose uniformly:

```text
input ──► runnable ──► output
```

Every Runnable has the same three methods:

```ts
await chain.invoke(input);    // one input → one output (most common)
await chain.stream(input);    // stream output as it's produced
await chain.batch(inputs);    // many inputs in parallel
```

Compose them with `.pipe()` — the output of one feeds the next. This builds a `RunnableSequence` (a graph of runnables):

```ts
const chain = prompt.pipe(model).pipe(parser);   // prompt → model → parser
await chain.invoke({ topic: "RAG" });
```

Most LangChain components are runnables composed this way.

---

## Composition primitives

### `.pipe()` vs `.assign()`

- **`.pipe()` replaces** — the previous step's output becomes the next step's input. The value is reshaped; earlier data is gone unless you carried it.
- **`.assign()` augments** — keeps the whole input object and adds keys to it. Each assigned runnable receives the same input; its result is merged in under a key. Nothing is lost.

```ts
// pipe: shape changes, question is gone after retrieve
question ──pipe──► retrieve ──► Document[]        // ❌ prompt can't see the question anymore

// assign: shape grows, question kept alongside the new context
{ question } ──assign({ context: retrieve })──► { question, context }   // ✅ both available
```

Rule of thumb: **`pipe` for a linear transform, `assign` when a later step needs both the original input and the new value** — which is most RAG and agent prompts (they need the question *and* the retrieved context).

### RunnableLambda — your business logic as a step

Wrap any function so it becomes a Runnable you can `.pipe()` into a chain. Where DB lookups, transforms, and side logic live.

```ts
import { RunnableLambda } from "@langchain/core/runnables";

const loadUser = RunnableLambda.from(async (userId: string) =>
  db.users.findById(userId),
);

const chain = loadUser.pipe(profilePrompt).pipe(model);
```

### RunnablePassthrough.assign — enrich state without losing it

Carry the input forward and add keys to it instead of replacing it. Common in RAG and agent pipelines, where you accumulate context.

```ts
import { RunnablePassthrough } from "@langchain/core/runnables";

const enriched = RunnablePassthrough.assign({
  customer: loadCustomer,    // each runs on the input, result merged in under this key
  orders: loadOrders,
});

// { customerId: "123" }  →  { customerId: "123", customer: {...}, orders: [...] }
```

### RunnableParallel — run branches simultaneously

Fan one input out to multiple runnables at once, collect a keyed object. Common in extraction.

```ts
import { RunnableParallel } from "@langchain/core/runnables";

const extract = RunnableParallel.from({
  summary: summaryChain,
  sentiment: sentimentChain,
  entities: entityChain,
});

// document  →  { summary: "...", sentiment: "...", entities: [...] }
const out = await extract.invoke(document);
```

---

## Chat model call

```ts
import { ChatOpenAI } from "@langchain/openai";

const model = new ChatOpenAI({ model: "gpt-4o-mini" });

const res = await model.invoke("Explain RAG in one sentence.");
console.log(res.content);
```

## Prompt template

```ts
import { ChatPromptTemplate } from "@langchain/core/prompts";
import { ChatOpenAI } from "@langchain/openai";

const prompt = ChatPromptTemplate.fromMessages([
  ["system", "You are concise and practical."],
  ["human", "Explain {topic} in {words} words."],
]);

const chain = prompt.pipe(new ChatOpenAI({ model: "gpt-4o-mini" }));

const res = await chain.invoke({ topic: "embeddings", words: 40 });
console.log(res.content);
```

### Dedent multi-line prompts — `context` (LangChain v1)

LangChain v1 exports a **`context`** tagged template for authoring prompts: write the prompt indented to match your code, and `context` strips the shared leading indentation, trims outer whitespace, and aligns any multi-line `${...}` values — returning a plain string.

```ts
import { context } from "langchain";

const SYSTEM_PROMPT = context`
  You are a rigorous, execution-focused agent.

  Rules:
  - Prefer evidence over assumptions.
  - State uncertainty explicitly.
`;   // → flush-left text, no leading code indentation
```

It's a **tagged template literal**: `` context`...` `` desugars to `context(strings, ...values)`, so the function receives the static chunks and any interpolations. A plain `` `...` `` template works too, but bakes the source indentation into the string.

## Messages & chat history

Models speak in **message objects**, not strings. The `["system", "..."]` tuple is shorthand; the classes give you control and are what you store and replay:

```ts
import { SystemMessage, HumanMessage, AIMessage } from "@langchain/core/messages";

await model.invoke([
  new SystemMessage("You are concise."),
  new HumanMessage("Explain RAG."),
]);
```

To inject prior turns into a prompt, leave a placeholder slot for them:

```ts
const prompt = ChatPromptTemplate.fromMessages([
  ["system", "You are a support agent."],
  ["placeholder", "{history}"],     // past messages drop in here (a MessagesPlaceholder)
  ["human", "{input}"],
]);

await prompt.pipe(model).invoke({
  history: [new HumanMessage("I was double charged"), new AIMessage("I can help.")],
  input: "When will I be refunded?",
});
```

> LangChain slots history *into a prompt* — it doesn't persist it. For durable multi-turn memory across requests, use LangGraph's checkpointer ([langgraph.md](langgraph.md)).

## Structured output

```ts
import { ChatOpenAI } from "@langchain/openai";
import * as z from "zod";

const Ticket = z.object({
  title: z.string(),
  priority: z.enum(["low", "medium", "high"]),
  labels: z.array(z.string()),
});

const model = new ChatOpenAI({ model: "gpt-4o-mini" });
const structured = model.withStructuredOutput(Ticket);

const ticket = await structured.invoke("Bug: checkout fails when card expires");
```

## Tools

```ts
import { tool } from "@langchain/core/tools";
import * as z from "zod";

const getWeather = tool(
  async ({ city }) => `Sunny in ${city}`,
  {
    name: "get_weather",
    description: "Get current weather for a city",
    schema: z.object({
      city: z.string().describe("City name"),
    }),
  },
);
```

## Agent

```ts
import { createAgent, tool } from "langchain";
import * as z from "zod";

const getWeather = tool(({ city }) => `Sunny in ${city}`, {
  name: "get_weather",
  description: "Get the weather for a city",
  schema: z.object({ city: z.string() }),
});

const agent = createAgent({
  model: "gpt-4o-mini",
  tools: [getWeather],
});

const res = await agent.invoke({
  messages: [{ role: "user", content: "Weather in Sydney?" }],
});
```

> LangChain's current JS overview uses `createAgent`, `tool`, and Zod schemas for tools.

---

## Routing

Pick one of several chains based on the input — a **single pass**, no loops or state, so it stays in LangChain (cyclic/stateful branching is where LangGraph comes in).

`RunnableBranch` — first matching condition wins, last entry is the default:

```ts
import { RunnableBranch } from "@langchain/core/runnables";

const route = RunnableBranch.from([
  [(x) => x.category === "billing", billingChain],
  [(x) => x.category === "refund", refundChain],
  generalChain,                                    // default — no condition
]);
```

Or a `RunnableLambda` that *returns* a runnable — LCEL invokes whatever you return (more flexible for dynamic choice):

```ts
const route = RunnableLambda.from((x) =>
  x.tokens > 4000 ? bigModelChain : smallModelChain,
);
```

Common shape — classify, then route on the result:

```ts
const chain = classify.pipe(route);   // classify → { category } → branch picks a chain
```

---

## When to move to LangGraph

Everything above is **single-flow**: `input → step1 → step2 → output`. Reach for [LangGraph](langgraph.md) the moment you need:

- **loops** (agent reasoning, retry-until-valid)
- **stateful or cyclic branching** (single-pass routing stays in LangChain — see [Routing](#routing))
- **persistent state** across steps or turns
- **human-in-the-loop** or **multi-agent** coordination

A LangGraph node is usually just a Runnable or an `async (state) => ({...})` — so the runnable knowledge above carries straight over.

```text
        START → classify ─┬─► research ─┐
                          ├─► summarise ─┼─► END
                          └─► escalate ──┘
```

---

## Output parsers

A raw model call returns an `AIMessage`. A parser is the last `.pipe()` step that turns it into what you actually want — a string, JSON, or a typed object.

```ts
import { StringOutputParser, JsonOutputParser } from "@langchain/core/output_parsers";

const text = await prompt.pipe(model).pipe(new StringOutputParser()).invoke(input);
const json = await prompt.pipe(model).pipe(new JsonOutputParser()).invoke(input);
```

For typed, schema-validated objects prefer `withStructuredOutput` (binds the schema *and* parses — see [Structured output](#structured-output) above) over a standalone parser:

```ts
const classifier = model.withStructuredOutput(Ticket);   // returns a typed object, validated
```

---

## RAG basics

Install common pieces:

```bash
npm i @langchain/openai @langchain/textsplitters
```

Split documents:

```ts
import { RecursiveCharacterTextSplitter } from "@langchain/textsplitters";

const splitter = new RecursiveCharacterTextSplitter({
  chunkSize: 800,
  chunkOverlap: 120,
});

const docs = await splitter.createDocuments([longText]);
```

Embed text:

```ts
import { OpenAIEmbeddings } from "@langchain/openai";

const embeddings = new OpenAIEmbeddings({
  model: "text-embedding-3-small",
});

const vector = await embeddings.embedQuery("refund policy");
```

Store and retrieve. **Vector store = storage; retriever = the retrieval interface.** `.asRetriever()` turns a store into a Runnable you can `.pipe()` into a chain — most chains consume a *retriever*, not a store directly.

```ts
import { MemoryVectorStore } from "langchain/vectorstores/memory";

const store = await MemoryVectorStore.fromDocuments(docs, embeddings);

// store API — returns Document[]
const results = await store.similaritySearch("refund policy", 4);

// retriever — a Runnable; composes into chains, returns Document[]
const retriever = store.asRetriever({ k: 4 });
const docs2 = await retriever.invoke("refund policy");
```

Wire retrieval into a chain with `RunnablePassthrough` (retrieve, keep the question, then answer):

```ts
const ragChain = RunnablePassthrough.assign({
  context: (x) => retriever.invoke(x.question),
}).pipe(answerPrompt).pipe(model);

await ragChain.invoke({ question: "What is the refund policy?" });
```

Answer with retrieved context (manual version):

```ts
const context = results.map((doc) => doc.pageContent).join("\n\n");

const res = await model.invoke([
  ["system", "Answer only from this context:\n\n" + context],
  ["human", "What is the refund policy?"],
]);
```

---

## Debugging

**`verbose: true`** — the quickest switch. Attaches a `ConsoleCallbackHandler` that prints every step (prompts, tool calls, outputs) to stdout:

```ts
const model = new ChatOpenAI({ model: "gpt-4o-mini", verbose: true });
// or per call:
await chain.invoke(input, { callbacks: [new ConsoleCallbackHandler()] });
//   import { ConsoleCallbackHandler } from "@langchain/core/tracers/console";
```

> Unlike Python, JS has no global `setVerbose`/`setDebug` — set `verbose` (or pass `callbacks`) per runnable.

**See the chain's structure** as a Mermaid diagram:

```ts
console.log(chain.getGraph().drawMermaid());   // paste into any Mermaid renderer
```

**Watch every step fire** — `streamEvents` emits an event per chain/model/tool start and per token. The main way to see *inside* a chain or agent:

```ts
for await (const ev of chain.streamEvents(input, { version: "v2" })) {
  if (ev.event === "on_chat_model_stream") process.stdout.write(ev.data.chunk.content);
  if (ev.event === "on_tool_start") console.log("tool:", ev.name, ev.data.input);
}
```

**Name steps** so traces and logs are readable:

```ts
chain.withConfig({ runName: "answer-question", tags: ["rag"] });
```

## LangSmith tracing

```bash
export LANGSMITH_TRACING=true
export LANGSMITH_API_KEY=lsv2_...
export LANGSMITH_PROJECT=my-project
```

LangChain automatically traces many runs when tracing env vars are set — full inputs, tool calls, and outputs per step.

---

## Practical recipes

**Make a one-shot classifier:**
```ts
const Category = z.object({
  category: z.enum(["billing", "bug", "sales", "other"]),
});

const classify = model.withStructuredOutput(Category);
await classify.invoke("I was charged twice.");
```

**Force plain string output from a prompt chain:**
```ts
const chain = prompt.pipe(model).pipe(new StringOutputParser());
```

**Pass runtime metadata for tracing:**
```ts
await chain.invoke(input, {
  tags: ["support"],
  metadata: { userId: "u_123" },
});
```

**Bind tools directly to a chat model:**
```ts
const modelWithTools = model.bindTools([getWeather]);
const res = await modelWithTools.invoke("What is the weather in Tokyo?");
```

**Add a timeout to a call:**
```ts
await model.invoke("Summarise this", { timeout: 10_000 });
```

**Retry flaky model calls:**
```ts
const reliable = model.withRetry({ stopAfterAttempt: 3 });
await reliable.invoke("Write release notes");
```

**Fall back to another model on error:**
```ts
const llm = primary.withFallbacks({ fallbacks: [backupModel] });
await llm.invoke(prompt);   // tries primary, then backupModel on failure
```

**Read token usage for cost tracking:**
```ts
const res = await model.invoke("Summarise this");
res.usage_metadata;   // { input_tokens, output_tokens, total_tokens }
```

**Stream tokens to stdout:**
```ts
for await (const chunk of await chain.stream(input)) {
  process.stdout.write(String(chunk.content ?? chunk));
}
```

**Insert a transform mid-chain (RunnableLambda):**
```ts
const chain = retriever
  .pipe(RunnableLambda.from((docs) => docs.map((d) => d.pageContent).join("\n\n")))
  .pipe(answerPrompt).pipe(model);
```

**Load `.env` in local scripts:**
```bash
npm i dotenv
node --env-file=.env src/index.js
```

---

## Tips

- **Everything is a Runnable** with `invoke`/`stream`/`batch` — learn that interface first; the rest is composition.
- `RunnablePassthrough.assign` to *enrich* state, `RunnableLambda` to inject logic, `RunnableParallel` to fan out — the three you reach for constantly.
- Use Zod schemas for tool inputs and structured output; prefer `withStructuredOutput` over standalone parsers.
- LangChain for single-flow pipelines; **LangGraph the moment you need loops, branching, state, or multi-agent.**
- Keep retrievers and model calls behind small functions so tests can stub them.
- Debug with `streamEvents` (see inside a run) and `getGraph().drawMermaid()` (see the structure); name steps with `withConfig({ runName })`.
- Stateless routing (`RunnableBranch`) stays in LangChain; cyclic/stateful branching is LangGraph.
- Trace early; agent bugs are usually invisible without inputs, tool calls, and outputs.
- Pin package versions in production. LangChain APIs move.
