# LangChain

**What it is:** A JavaScript/TypeScript framework for building LLM apps with models, prompts, tools, retrievers, and agents.

**Why people use it:** It gives standard building blocks for wiring LLMs to real data and actions without rewriting the same glue code.

**Typically used for:** Agents, tool calling, RAG pipelines, chatbots, structured extraction, model-provider switching, and tracing/evaluation workflows.

> This sheet focuses on LangChain JS/TS. Source: https://docs.langchain.com/oss/javascript/langchain/overview

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

## Most common commands

```bash
npm i langchain @langchain/core zod
npm i @langchain/openai @langchain/anthropic
npm i @langchain/community
npm i @langchain/textsplitters
npm i @langchain/langgraph
```

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

## LCEL chains

LCEL means `runnable.pipe(nextRunnable)`.

```ts
import { StringOutputParser } from "@langchain/core/output_parsers";

const chain = prompt
  .pipe(model)
  .pipe(new StringOutputParser());

const text = await chain.invoke({ topic: "LangChain", words: 30 });
```

Parallel steps:

```ts
import { RunnableParallel } from "@langchain/core/runnables";

const chain = RunnableParallel.from({
  joke: jokePrompt.pipe(model),
  haiku: haikuPrompt.pipe(model),
});

const out = await chain.invoke({ topic: "databases" });
```

Streaming:

```ts
const stream = await model.stream("Write a short haiku about logs.");

for await (const chunk of stream) {
  process.stdout.write(String(chunk.content));
}
```

Batching:

```ts
const results = await model.batch([
  "Summarise Redis",
  "Summarise Postgres",
  "Summarise Kafka",
]);
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

Simple in-memory vector store:

```ts
import { MemoryVectorStore } from "langchain/vectorstores/memory";

const store = await MemoryVectorStore.fromDocuments(docs, embeddings);
const results = await store.similaritySearch("refund policy", 4);
```

Answer with retrieved context:

```ts
const context = results.map((doc) => doc.pageContent).join("\n\n");

const res = await model.invoke([
  ["system", "Answer only from this context:\n\n" + context],
  ["human", "What is the refund policy?"],
]);
```

---

## LangSmith tracing

```bash
export LANGSMITH_TRACING=true
export LANGSMITH_API_KEY=lsv2_...
export LANGSMITH_PROJECT=my-project
```

LangChain automatically traces many runs when tracing env vars are set.

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

**Load `.env` in local scripts:**
```bash
npm i dotenv
node --env-file=.env src/index.js
```

---

## Tips

- Use Zod schemas for tool inputs and structured output.
- Use LCEL for simple pipelines; use LangGraph when control flow becomes stateful or cyclic.
- Keep retrievers and model calls behind small functions so tests can stub them.
- Trace early; agent bugs are usually invisible without inputs, tool calls, and outputs.
- Pin package versions in production. LangChain APIs move.
