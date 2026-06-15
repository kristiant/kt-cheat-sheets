# LangGraph

**What it is:** A low-level orchestration framework for building stateful, multi-step LLM agents as graphs.

**Why people use it:** It makes agent control flow explicit: nodes update shared state, edges decide what runs next, and checkpoints make runs durable.

**Typically used for:** Tool-calling agents, multi-step workflows, human-in-the-loop flows, long-running agents, memory, retries, and cyclic reasoning loops.

> **LangGraph is the orchestration layer over [LangChain](langchain.md)'s building blocks.** LangChain handles single-flow (`input → step → output`); LangGraph adds loops, branching, persistent state, and multi-agent flow. A node is usually just a LangChain Runnable or an `async (state) => ({...})` — so the runnable knowledge carries straight over. For higher-level *patterns* (reliability, cost, multi-agent topologies) see [agentic-patterns.md](agentic-patterns.md).
> Source: https://docs.langchain.com/oss/javascript/langgraph/overview

---

## Setup

```bash
npm i @langchain/langgraph @langchain/core zod
npm i @langchain/openai
```

```bash
export OPENAI_API_KEY=sk-...
```

---

## Mental model — State, Nodes, Edges

A LangGraph app is three primitives:

| Primitive | What it is |
|---|---|
| **State** | A typed shared object every node reads and writes. Defined with `Annotation.Root`. **Reducers** decide how writes merge (required once nodes run in parallel). |
| **Node** | A function (or Runnable) `(state) => partialUpdate`. The work. Returns *only the keys it changes*. |
| **Edge** | Wiring that decides what runs next — a fixed edge, or a **conditional edge** whose router function returns the next node's name. |

The build lifecycle is always the same shape:

```ts
const graph = new StateGraph(State)   // 1. declare the state shape
  .addNode("work", workFn)            // 2. add nodes (the work)
  .addEdge(START, "work")             // 3. wire the flow: START → ... → END
  .addEdge("work", END)
  .compile();                         // 4. produce a runnable graph

await graph.invoke(input);            // 5. run it — invoke / stream / batch
```

> A compiled graph is **itself a Runnable** (same `invoke`/`stream`/`batch` interface as anything in [LangChain](langchain.md)) — so it nests inside other chains and graphs as a node.

Common imports:

```ts
import { Annotation, StateGraph, START, END, MemorySaver } from "@langchain/langgraph";
```

---

## Minimal graph

```ts
import { Annotation, END, START, StateGraph } from "@langchain/langgraph";

const State = Annotation.Root({
  topic: Annotation<string>(),
  joke: Annotation<string>(),
});

const graph = new StateGraph(State)
  .addNode("writeJoke", async (state) => ({
    joke: `A short joke about ${state.topic}`,
  }))
  .addEdge(START, "writeJoke")
  .addEdge("writeJoke", END)
  .compile();

const result = await graph.invoke({ topic: "databases" });
console.log(result.joke);
```

## State with reducers

A reducer decides how a key's updates merge. Without one, the last write **overwrites**; with an append reducer, writes accumulate. This is what makes parallel fan-out safe — two nodes writing the same key in one step would otherwise conflict.

```ts
const State = Annotation.Root({
  messages: Annotation<string[]>({
    reducer: (left, right) => left.concat(right),   // append, don't overwrite
    default: () => [],
  }),
});
```

> `MessagesAnnotation` is a prebuilt state with exactly this messages reducer — use it for chat agents instead of redefining it.

## Conditional edges

```ts
const shouldContinue = (state: { count: number }) => {
  return state.count >= 3 ? END : "increment";
};

const graph = new StateGraph(State)
  .addNode("increment", async (state) => ({ count: state.count + 1 }))
  .addEdge(START, "increment")
  .addConditionalEdges("increment", shouldContinue, ["increment", END])
  .compile();
```

## Chat state

```ts
import { MessagesAnnotation } from "@langchain/langgraph";
import { ChatOpenAI } from "@langchain/openai";

const model = new ChatOpenAI({ model: "gpt-4o-mini" });

const callModel = async (state: typeof MessagesAnnotation.State) => {
  const response = await model.invoke(state.messages);
  return { messages: [response] };
};

const graph = new StateGraph(MessagesAnnotation)
  .addNode("callModel", callModel)
  .addEdge(START, "callModel")
  .addEdge("callModel", END)
  .compile();
```

## Invoke

```ts
import { HumanMessage } from "@langchain/core/messages";

const result = await graph.invoke({
  messages: [new HumanMessage("Explain checkpointing")],
});
```

## Stream

```ts
const stream = await graph.stream(input, { streamMode: "updates" });

for await (const chunk of stream) {
  console.log(chunk);
}
```

---

## Tool-calling loop

```ts
import { AIMessage, ToolMessage } from "@langchain/core/messages";
import { tool } from "@langchain/core/tools";
import { END, MessagesAnnotation, START, StateGraph } from "@langchain/langgraph";
import { ChatOpenAI } from "@langchain/openai";
import * as z from "zod";

const add = tool(({ a, b }) => a + b, {
  name: "add",
  description: "Add two numbers",
  schema: z.object({ a: z.number(), b: z.number() }),
});

const toolsByName = { [add.name]: add };
const model = new ChatOpenAI({ model: "gpt-4o-mini" }).bindTools([add]);

const callModel = async (state: typeof MessagesAnnotation.State) => ({
  messages: [await model.invoke(state.messages)],
});

const callTools = async (state: typeof MessagesAnnotation.State) => {
  const last = state.messages.at(-1);
  if (!last || !AIMessage.isInstance(last)) return { messages: [] };

  const messages: ToolMessage[] = [];
  for (const toolCall of last.tool_calls ?? []) {
    messages.push(await toolsByName[toolCall.name].invoke(toolCall));
  }
  return { messages };
};

const route = (state: typeof MessagesAnnotation.State) => {
  const last = state.messages.at(-1);
  if (last && AIMessage.isInstance(last) && last.tool_calls?.length) {
    return "tools";
  }
  return END;
};

const agent = new StateGraph(MessagesAnnotation)
  .addNode("model", callModel)
  .addNode("tools", callTools)
  .addEdge(START, "model")
  .addConditionalEdges("model", route, ["tools", END])
  .addEdge("tools", "model")
  .compile();
```

> LangGraph's JS quickstart uses `StateGraph`, `START`, `END`, conditional edges, and explicit model/tool nodes.

---

## Memory / checkpointing

Short-term memory needs a checkpointer and a stable `thread_id`.

```ts
import { MemorySaver } from "@langchain/langgraph";

const checkpointer = new MemorySaver();

const graph = builder.compile({ checkpointer });

await graph.invoke(
  { messages: [new HumanMessage("My name is Ada")] },
  { configurable: { thread_id: "user-123" } },
);

await graph.invoke(
  { messages: [new HumanMessage("What is my name?")] },
  { configurable: { thread_id: "user-123" } },
);
```

> LangGraph documents short-term memory as part of agent state and long-term memory for user/application data.

## Interrupts / human review

```ts
import { interrupt } from "@langchain/langgraph";

const review = async (state: typeof MessagesAnnotation.State) => {
  const approved = interrupt({
    question: "Approve sending this response?",
    draft: state.messages.at(-1)?.content,
  });

  return { approved };
};
```

Resume with a command:

```ts
import { Command } from "@langchain/langgraph";

await graph.invoke(new Command({ resume: true }), config);
```

---

## Functional API

Use this when a loop is easier to read as a function.

```ts
import { entrypoint, task } from "@langchain/langgraph";

const callModel = task({ name: "callModel" }, async (input: string) => {
  return model.invoke(input);
});

const app = entrypoint({ name: "app" }, async (input: string) => {
  const result = await callModel(input);
  return result.content;
});

await app.invoke("Summarise LangGraph");
```

---

## Practical recipes

**Create a graph with one model node:**
```ts
const graph = new StateGraph(MessagesAnnotation)
  .addNode("model", callModel)
  .addEdge(START, "model")
  .addEdge("model", END)
  .compile();
```

**Route to a node based on state:**
```ts
.addConditionalEdges("classify", (state) => state.needsSearch ? "search" : END, [
  "search",
  END,
])
```

**Append messages instead of replacing them:**
```ts
import { MessagesAnnotation } from "@langchain/langgraph";
```

**Persist a conversation thread:**
```ts
const config = { configurable: { thread_id: "tenant:user:123" } };
await graph.invoke(input, config);
```

**Debug node-by-node output:**
```ts
for await (const update of await graph.stream(input, { streamMode: "updates" })) {
  console.dir(update, { depth: null });
}
```

**Add tracing metadata:**
```ts
await graph.invoke(input, {
  tags: ["agent"],
  metadata: { userId: "u_123" },
});
```

---

## Tips

- Use LangChain agents first if you only need a standard tool loop.
- Use LangGraph when you need explicit branches, cycles, state, memory, or human review.
- Keep node functions small and deterministic where possible.
- Make route functions boring: return node names or `END`.
- Use stable `thread_id` values for memory; changing them creates separate conversations.
- Stream `updates` while developing so you can see which node changed state.
