# LangGraph

**What it is:** A low-level orchestration framework for building stateful, multi-step LLM agents as graphs.

**Why people use it:** It makes agent control flow explicit: nodes update shared state, edges decide what runs next, and checkpoints make runs durable.

**Typically used for:** Tool-calling agents, multi-step workflows, human-in-the-loop flows, long-running agents, memory, retries, and cyclic reasoning loops.

> This sheet focuses on LangGraph JS/TS. Source: https://docs.langchain.com/oss/javascript/langgraph/overview

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

## Most common imports

```ts
import {
  Annotation,
  END,
  MemorySaver,
  START,
  StateGraph,
} from "@langchain/langgraph";
```

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

Reducers merge updates from multiple nodes.

```ts
const State = Annotation.Root({
  messages: Annotation<string[]>({
    reducer: (left, right) => left.concat(right),
    default: () => [],
  }),
});
```

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
