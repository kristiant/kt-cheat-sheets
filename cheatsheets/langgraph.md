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

---

## Real-world patterns (sourced from open-source TypeScript projects)

### Tool approval as an explicit node — `interrupt()` inside a tool

Rather than calling `interrupt()` in a node, inject it into the tool itself. The tool pauses mid-execution; the graph replays cleanly when resumed.

Seen in: [`agentailor/fullstack-langgraph-nextjs-agent`](https://github.com/agentailor/fullstack-langgraph-nextjs-agent) — `src/lib/agent/builder.ts`

```ts
// Tool that requires human approval before executing
const deleteRecord = tool(
  async (args) => {
    const decision = interrupt({
      type: "approval_required",
      message: `Delete record "${args.id}"? This cannot be undone.`,
      args,
    });
    if (decision !== "yes") return `Deletion of "${args.id}" was rejected.`;
    return `Record "${args.id}" deleted successfully.`;
  },
  {
    name: "delete_record",
    description: "Delete a record by ID. Requires human approval.",
    schema: z.object({ id: z.string() }),
  }
);
```

Resume after the interrupt:

```ts
// Client receives the interrupt payload, shows UI, then resumes
await graph.invoke(
  new Command({ resume: "yes" }),   // or "no"
  { configurable: { thread_id } },
);
```

### Tool approval as a dedicated graph node

For finer control — approve any tool call before `ToolNode` executes it. Adds an explicit `tool_approval` node between `agent` and `tools`.

Seen in: [`agentailor/fullstack-langgraph-nextjs-agent`](https://github.com/agentailor/fullstack-langgraph-nextjs-agent)

```ts
import {
  StateGraph, MessagesAnnotation, START, END,
  interrupt, Command,
} from "@langchain/langgraph";
import { ToolNode } from "@langchain/langgraph/prebuilt";
import { ToolMessage } from "@langchain/core/messages";
import type { ToolCall } from "@langchain/core/messages/tool";

class AgentBuilder {
  private async approveToolCall(state: typeof MessagesAnnotation.State) {
    const lastMessage = state.messages.at(-1)!;
    const toolCall = (lastMessage as any).tool_calls?.at(-1) as ToolCall;

    const review = interrupt<
      { question: string; toolCall: ToolCall },
      { action: "continue" | "update" | "feedback"; data: unknown }
    >({ question: "Is this correct?", toolCall });

    if (review.action === "continue") return new Command({ goto: "tools" });

    if (review.action === "update") {
      // Human edited the args before proceeding
      return new Command({
        goto: "tools",
        update: { messages: [{ ...lastMessage, tool_calls: [{ ...toolCall, args: review.data }] }] },
      });
    }

    // "feedback" — return a synthetic ToolMessage and loop back to agent
    return new Command({
      goto: "agent",
      update: {
        messages: [new ToolMessage({ name: toolCall.name, content: review.data as string, tool_call_id: toolCall.id! })],
      },
    });
  }

  build() {
    return new StateGraph(MessagesAnnotation)
      .addNode("agent", this.callModel.bind(this))
      .addNode("tools", new ToolNode(this.tools))
      .addNode("tool_approval", this.approveToolCall.bind(this), {
        ends: ["tools", "agent"],   // pre-declare reachable nodes
      })
      .addEdge(START, "agent")
      .addConditionalEdges("agent", (s) =>
        (s.messages.at(-1) as any)?.tool_calls?.length ? "tool_approval" : END,
      )
      .addEdge("tools", "agent")
      .compile({ checkpointer: this.checkpointer });
  }
}
```

### PostgresSaver with lazy loading and MemorySaver fallback

Don't hard-require the Postgres driver — load it lazily so the in-memory path has zero overhead.

Seen in: [`ac12644/langgraph-starter-kit`](https://github.com/ac12644/langgraph-starter-kit) — `src/config/checkpointer.ts`

```ts
import { MemorySaver, InMemoryStore } from "@langchain/langgraph";
import type { BaseCheckpointSaver, BaseStore } from "@langchain/langgraph-checkpoint";

export async function getCheckpointer(): Promise<BaseCheckpointSaver> {
  if (!process.env.DATABASE_URL) return new MemorySaver();

  let PostgresSaver: typeof import("@langchain/langgraph-checkpoint-postgres").PostgresSaver;
  try {
    ({ PostgresSaver } = await import("@langchain/langgraph-checkpoint-postgres"));
  } catch (err) {
    const code = (err as { code?: string }).code;
    if (code === "ERR_MODULE_NOT_FOUND" || code === "MODULE_NOT_FOUND") {
      throw new Error("DATABASE_URL is set but @langchain/langgraph-checkpoint-postgres is not installed.");
    }
    throw err;
  }

  const saver = PostgresSaver.fromConnString(process.env.DATABASE_URL);
  await saver.setup(); // creates tables — idempotent
  return saver;
}

export function getStore(): BaseStore {
  return new InMemoryStore(); // swap for persistent store if needed
}
```

### Multi-agent Supervisor pattern

Route between specialised agents using `@langchain/langgraph-supervisor`. Each agent is a compiled graph; the supervisor decides who handles each step.

Seen in: [`ac12644/langgraph-starter-kit`](https://github.com/ac12644/langgraph-starter-kit) — `src/apps/support.ts`

```bash
npm i @langchain/langgraph-supervisor
```

```ts
import { createReactAgent } from "@langchain/langgraph/prebuilt";
import { createSupervisor } from "@langchain/langgraph-supervisor";

const billingAgent = createReactAgent({
  name: "billing_agent",
  llm,
  tools: [lookupCustomer, checkBalance, issueRefund, escalateToHuman],
  prompt: "You are a billing specialist. Handle account lookups, balance inquiries, and refunds.",
});

const techAgent = createReactAgent({
  name: "tech_support_agent",
  llm,
  tools: [lookupOrder, createTicket, listTickets, escalateToHuman],
  prompt: "You are a technical support specialist. Handle order issues and bug reports.",
});

const returnsAgent = createReactAgent({
  name: "returns_agent",
  llm,
  tools: [lookupOrder, initiateReturn, escalateToHuman],
  prompt: "You are a returns specialist. Handle returns and exchanges for delivered orders.",
});

const supportApp = createSupervisor({
  agents: [billingAgent, techAgent, returnsAgent],
  llm,
  outputMode: "last_message",
  supervisorName: "support_router",
  prompt: [
    "Route to billing_agent for payments/refunds,",
    "tech_support_agent for orders/bugs,",
    "returns_agent for returns.",
    "Ask a clarifying question if intent is unclear.",
  ].join(" "),
}).compile({ checkpointer: await getCheckpointer() });
```

### Swarm pattern — peer-to-peer agent handoff

Agents hand off to each other directly (no central supervisor). Each agent decides who should handle the next step.

Seen in: [`ac12644/langgraph-starter-kit`](https://github.com/ac12644/langgraph-starter-kit) — `src/agents/swarm.ts`

```bash
npm i @langchain/langgraph-swarm
```

```ts
import { Annotation, MessagesAnnotation } from "@langchain/langgraph";
import { createSwarm } from "@langchain/langgraph-swarm";

// Extend MessagesAnnotation to track which agent is active
const SwarmState = Annotation.Root({
  ...MessagesAnnotation.spec,
  activeAgent: Annotation<string>(),
});

const swarmApp = createSwarm({
  agents: [agentA, agentB, agentC],
  defaultActiveAgent: "agentA",
  stateSchema: SwarmState,
}).compile({ checkpointer: await getCheckpointer() });
```

### MCP tools from a database — dynamic tool loading

Load enabled MCP servers from a DB at runtime and pass their tools into the agent.

Seen in: [`agentailor/fullstack-langgraph-nextjs-agent`](https://github.com/agentailor/fullstack-langgraph-nextjs-agent) — `src/lib/agent/mcp.ts`

```bash
npm i @langchain/mcp-adapters
```

```ts
import { MultiServerMCPClient } from "@langchain/mcp-adapters";

export async function getMCPTools() {
  // Load server configs from DB (or config file)
  const servers = await prisma.mCPServer.findMany({ where: { enabled: true } });

  const mcpServers = Object.fromEntries(
    servers.map((s) => [
      s.name,
      s.type === "stdio"
        ? { transport: "stdio", command: s.command, args: s.args }
        : { transport: "http", url: s.url, headers: s.headers },
    ])
  );

  const client = new MultiServerMCPClient({
    mcpServers,
    throwOnLoadError: false,
    prefixToolNameWithServerName: true,   // prevents name collisions across servers
  });

  return client.getTools();
}

// Wire into graph
const tools = await getMCPTools();
const agent = new AgentBuilder({ tools, llm, prompt, checkpointer }).build();
```

### Custom PostgresSaver with SSL — production connection string

```ts
import { PostgresSaver } from "@langchain/langgraph-checkpoint-postgres";

const saver = PostgresSaver.fromConnString(
  `${process.env.DATABASE_URL}?sslmode=${process.env.DB_SSLMODE ?? "require"}`
);
await saver.setup();
```

### Ingestion + retrieval split graphs — RAG chatbot

Two separate compiled graphs — one for document ingestion, one for question answering. Neither knows about the other; they share the same vector store.

Seen in: [`mayooear/ai-pdf-chatbot-langchain`](https://github.com/mayooear/ai-pdf-chatbot-langchain) — `src/ingestion_graph.ts`, `src/retrieval_graph.ts`

```
src/
  ingestion_graph.ts   → upload → chunk → embed → store
  retrieval_graph.ts   → query → retrieve → generate → stream
  shared/configuration.ts   → shared model + vector store config
```

The ingestion graph is a one-shot pipeline (no cycles); the retrieval graph is a ReAct loop that routes to a retrieval node when context is needed.

### Zod-based state definition

Import `@langchain/langgraph/zod` to unlock a `.langgraph.reducer()` method on any Zod field. Clean alternative to `Annotation.Root` — state is co-located with its validation schema.

Seen in: [`langchain-ai/agents-from-scratch-ts`](https://github.com/langchain-ai/agents-from-scratch-ts) — `src/schemas.ts`

```ts
import { z } from "zod";
import { BaseMessage } from "@langchain/core/messages";
import "@langchain/langgraph/zod";   // augments Zod with .langgraph.reducer()
import { addMessages, Messages, StateGraph } from "@langchain/langgraph";

const State = z.object({
  messages: z.custom<BaseMessage[]>()
    .default(() => [])
    .langgraph.reducer<Messages>((left, right) => addMessages(left, right)),
  classification: z.enum(["ignore", "respond", "notify"]).nullable().default(null),
  email_input: z.any(),
});

// Feed the schema directly into StateGraph — same API as Annotation.Root
const graph = new StateGraph(State)
  .addNode("router", routerNode)
  .addEdge(START, "router")
  .compile();

// Inferred types work as expected
type StateType = z.infer<typeof State>;
```

### Nested subgraph as a node

Compile an inner graph and insert it directly as a node in the outer graph. The inner graph runs to completion (including its own interrupts and loops) each time that node is reached.

Seen in: [`langchain-ai/agents-from-scratch-ts`](https://github.com/langchain-ai/agents-from-scratch-ts) — `src/email_assistant_hitl.ts`

```ts
// Inner graph — handles its own llm ↔ tool loop
const agentBuilder = new StateGraph(EmailAgentHITLState)
  .addNode("llm_call", llmCallNode)
  .addNode("interrupt_handler", interruptHandlerNode)
  .addEdge(START, "llm_call")
  .addConditionalEdges("llm_call", shouldContinue, {
    interrupt_handler: "interrupt_handler",  // named map for explicit routing
    [END]: END,
  })
  .addEdge("interrupt_handler", "llm_call");

const responseAgent = agentBuilder.compile();  // compiled subgraph

// Outer graph — uses the subgraph as a node
const emailAssistant = new StateGraph(EmailAgentHITLState)
  .addNode("triage_router", triageRouterNode)
  .addNode("response_agent", responseAgent)  // subgraph inserted here
  .addEdge(START, "triage_router")
  .addConditionalEdges("triage_router",
    (state) => state.classification === "respond" ? "response_agent" : END,
    { response_agent: "response_agent", [END]: END },
  )
  .addEdge("response_agent", END)
  .compile();
```

State keys must overlap for values to pass in/out. The subgraph's interrupts surface to the outer graph's caller.

### `HumanInterrupt` — structured prebuilt interrupt schema

Use `HumanInterrupt` / `HumanResponse` from `@langchain/langgraph/prebuilt` instead of raw `interrupt()` so Agent Chat UI can auto-render the review panel.

Seen in: [`langchain-ai/langgraphjs-gen-ui-examples`](https://github.com/langchain-ai/langgraphjs-gen-ui-examples) (email agent), [`langchain-ai/agents-from-scratch-ts`](https://github.com/langchain-ai/agents-from-scratch-ts)

```ts
import { interrupt } from "@langchain/langgraph";
import { HumanInterrupt, HumanResponse } from "@langchain/langgraph/prebuilt";

// Pause with a structured interrupt — Agent Chat UI reads config to render buttons
const humanReview = interrupt<HumanInterrupt[], HumanResponse[]>([{
  action_request: {
    action: "Review email draft",   // shown as the title
    args: { subject, body, to },    // shown as the detail fields
  },
  description: "# Email Draft\n...",  // markdown rendered in the panel
  config: {
    allow_ignore: true,   // shows "Ignore" button
    allow_respond: true,  // shows free-text input
    allow_edit: true,     // shows editable args form
    allow_accept: true,   // shows "Accept & Send" button
  },
}])[0];

// humanReview.type ∈ "ignore" | "response" | "edit" | "accept"
// humanReview.args = the user's input (string for response, object for edit)

switch (humanReview.type) {
  case "accept":
    await sendEmail(humanReview.args as EmailArgs);
    break;
  case "edit":
    await sendEmail(humanReview.args as EmailArgs);  // edited args
    break;
  case "response":
    // Pass feedback back to LLM to rewrite
    return { messages: [{ role: "tool", content: `Feedback: ${humanReview.args}`, tool_call_id }] };
  case "ignore":
    return new Command({ goto: END });
}
```

### Extraction node — structured data from conversation into typed state

Use tool-calling to extract structured fields from the conversation into typed state. Route to the main agent node only once all required fields are present.

Seen in: [`langchain-ai/langgraphjs-gen-ui-examples`](https://github.com/langchain-ai/langgraphjs-gen-ui-examples) — trip planner `nodes/extraction.ts`

```ts
export async function extractionNode(state: StateType) {
  const schema = z.object({
    location: z.string().describe("The destination city or country"),
    startDate: z.string().optional().describe("Start date YYYY-MM-DD"),
    numberOfGuests: z.number().describe("Number of guests, default 2"),
  });

  const model = new ChatOpenAI({ model: "gpt-4o", temperature: 0 }).bindTools([
    { name: "extract", description: "Extract trip details", schema },
  ]);

  const response = await model.invoke([
    { role: "system", content: EXTRACTION_PROMPT },
    { role: "human", content: formatMessages(state.messages) },
  ]);

  const toolCall = response.tool_calls?.[0];
  if (!toolCall) {
    // No tool call = missing required fields; model responded with a clarifying question
    return { messages: [response] };
  }

  // Write extracted data into a dedicated state field
  return {
    tripDetails: { ...toolCall.args, numberOfGuests: toolCall.args.numberOfGuests ?? 2 },
    messages: [response, { role: "tool", content: "Extracted", tool_call_id: toolCall.id }],
  };
}

// Graph routes: if tripDetails is still undefined after extraction, go back to END
// (the model asked for clarification); otherwise proceed to the main agent
const graph = new StateGraph(State)
  .addNode("extraction", extractionNode)
  .addNode("callTools", callToolsNode)
  .addConditionalEdges("extraction",
    (state) => state.tripDetails ? "callTools" : END,
    ["callTools", END],
  )
  .compile();
```

### Cross-thread memory with `BaseStore`

`BaseStore` persists data across threads (unlike checkpointer, which is per-thread). Inject it at compile time; access it in node functions via `config.store` or pass it explicitly.

Seen in: [`langchain-ai/agents-from-scratch-ts`](https://github.com/langchain-ai/agents-from-scratch-ts) — `src/email_assistant_hitl_memory.ts`

```ts
import { InMemoryStore, BaseStore } from "@langchain/langgraph";

// Namespaces are hierarchical string arrays — like folder paths
const TRIAGE_NS = ["email_assistant", "triage_preferences"];
const RESPONSE_NS = ["email_assistant", "response_preferences"];

async function getMemory(store: BaseStore, namespace: string[], defaultContent = "") {
  const item = await store.get(namespace, "user_preferences");
  if (item) return item.value.memoryContent as string;
  // Bootstrap on first access
  await store.put(namespace, "user_preferences", { memoryContent: defaultContent });
  return defaultContent;
}

async function updateMemory(store: BaseStore, namespace: string[], messages: BaseMessage[]) {
  const existing = await store.get(namespace, "user_preferences");
  const currentProfile = existing?.value.memoryContent ?? "";
  // Pass existing profile + new messages to LLM → condensed updated profile
  const updatedProfile = await summariseWithLLM(currentProfile, messages);
  await store.put(namespace, "user_preferences", { memoryContent: updatedProfile });
}

// In a node function — store is available via config
const triageRouterNode = async (state: StateType, config: LangGraphRunnableConfig) => {
  const store = config.store as BaseStore;
  const triagePreferences = await getMemory(store, TRIAGE_NS, defaultTriageInstructions);
  // Inject stored preferences into the system prompt
  const systemPrompt = BASE_PROMPT.replace("{triage_instructions}", triagePreferences);
  // ...
};

// Wire it in at compile time
const store = new InMemoryStore();  // swap for PostgresStore in prod
const app = graph.compile({ checkpointer, store });
await app.invoke(input, { configurable: { thread_id: "user-123" } });
// store persists across all threads — updates accumulate user preferences over time
```

### Configurable agent — runtime model + prompt selection

Pass per-call config via `config.configurable` to let callers pick the model and system prompt at runtime without rebuilding the graph.

Seen in: [`langchain-ai/fullstack-chat-server`](https://github.com/langchain-ai/fullstack-chat-server) — `src/react_agent/configuration.ts`

```ts
import { Annotation } from "@langchain/langgraph";
import { RunnableConfig } from "@langchain/core/runnables";
import { initChatModel } from "langchain/chat_models/universal";

// Declare a separate config schema (not state) — passed as second arg to StateGraph
export const ConfigurationSchema = Annotation.Root({
  systemPromptTemplate: Annotation<string>,
  model: Annotation<string>,           // e.g. "anthropic/claude-3-5-sonnet-20240620"
});

export function ensureConfiguration(config: RunnableConfig) {
  const c = config.configurable ?? {};
  return {
    systemPromptTemplate: c.systemPromptTemplate ?? DEFAULT_PROMPT,
    model: c.model ?? "anthropic/claude-3-5-sonnet-20240620",
  };
}

// Use as the second arg — config values are read per-invocation, not baked in at compile time
const workflow = new StateGraph(MessagesAnnotation, ConfigurationSchema)
  .addNode("callModel", async (state, config) => {
    const { model, systemPromptTemplate } = ensureConfiguration(config);
    // initChatModel parses "provider/model" strings automatically
    const llm = await initChatModel(model.split("/")[1], {
      modelProvider: model.split("/")[0],
    });
    const response = await llm.bindTools(TOOLS).invoke([
      { role: "system", content: systemPromptTemplate },
      ...state.messages,
    ]);
    return { messages: [response] };
  })
  // ...
  .compile();

// Caller overrides model at runtime — no graph rebuild needed
await workflow.invoke(input, {
  configurable: {
    thread_id: "user-123",
    model: "openai/gpt-4o",
    systemPromptTemplate: "You are a concise assistant.",
  },
});
```

### LangGraph Platform auth — JWT via Supabase

`@langchain/langgraph-sdk/auth` exposes an `Auth` class for request-level JWT validation. Wired at deployment time; every run is scoped to the authenticated user.

Seen in: [`langchain-ai/fullstack-chat-server`](https://github.com/langchain-ai/fullstack-chat-server) — `src/react_agent/auth.ts`

```bash
# langgraph.json — tell LangGraph Platform where auth lives
# { "auth": { "path": "./src/react_agent/auth.ts:auth" } }
```

```ts
import { Auth, HTTPException } from "@langchain/langgraph-sdk/auth";
import { createClient } from "@supabase/supabase-js";

const supabase = createClient(process.env.SUPABASE_URL!, process.env.SUPABASE_ANON_KEY!);

export const auth = new Auth()
  .authenticate(async (request: Request) => {
    const authHeader = request.headers.get("authorization");
    if (!authHeader) throw new HTTPException(401, { message: "Missing Authorization header" });

    const token = authHeader.replace("Bearer ", "");
    const { data, error } = await supabase.auth.getUser(token);
    if (error || !data?.user) throw new HTTPException(401, { message: "Invalid token" });

    return {
      identity: data.user.id,
      permissions: [
        "threads:create", "threads:read", "threads:create_run",
        "assistants:read",
      ],
      display_name: data.user.email ?? data.user.id,
    };
  });
```

### Generative UI — stream React components from a node

Push typed UI components from inside a node using `typedUi()`. The client renders them alongside messages via `useStream`.

Seen in: [`langchain-ai/langgraphjs-gen-ui-examples`](https://github.com/langchain-ai/langgraphjs-gen-ui-examples) — `src/agent/stockbroker/nodes/tools.ts`

```bash
npm i @langchain/langgraph-sdk
```

```ts
// server — node function
import { typedUi } from "@langchain/langgraph-sdk/react-ui/server";
import type ComponentMap from "../../agent-uis/index"; // maps component names to props types
import type { LangGraphRunnableConfig } from "@langchain/langgraph";

export async function callTools(
  state: StockbrokerState,
  config: LangGraphRunnableConfig,
): Promise<StockbrokerUpdate> {
  const ui = typedUi<typeof ComponentMap>(config);   // typed to your component registry

  const message = await llm.bindTools(TOOLS).invoke(state.messages);

  const priceToolCall = message.tool_calls?.find((c) => c.name === "stock-price");
  if (priceToolCall) {
    const prices = await getPricesForTicker(priceToolCall.args.ticker);
    ui.push(
      { name: "stock-price", props: { ticker: priceToolCall.args.ticker, ...prices } },
      { message },  // links this UI component to the specific AIMessage
    );
  }

  return { messages: [message], ui: ui.items };
}
```

```tsx
// client — React with useStream from @langchain/langgraph-sdk/react
import { useStream } from "@langchain/langgraph-sdk/react";
import { uiMessageReducer } from "@langchain/langgraph-sdk/react-ui";

const stream = useStream<{ messages: Message[]; ui?: UIMessage[] }>({
  apiUrl: process.env.NEXT_PUBLIC_API_URL,
  assistantId: "stockbroker",
  threadId,
  onCustomEvent: (event, { mutate }) => {
    mutate((prev) => ({ ...prev, ui: uiMessageReducer(prev.ui ?? [], event) }));
  },
});
// stream.values.ui contains the pushed components — render them in a side panel
```

### SSE streaming with Fastify — messages-tuple mode

Stream LangGraph tokens to clients over Server-Sent Events. `streamMode: "messages"` gives `[chunk, metadata]` tuples with per-node attribution.

Seen in: [`ac12644/langgraph-starter-kit`](https://github.com/ac12644/langgraph-starter-kit) — `src/server/index.ts`

```ts
import { AIMessageChunk } from "@langchain/core/messages";

server.post("/:app/stream", async (req, reply) => {
  reply.raw.writeHead(200, {
    "Content-Type": "text/event-stream",
    "Cache-Control": "no-cache",
    Connection: "keep-alive",
  });

  const send = (event: { type: string; content?: unknown; node?: string }) => {
    reply.raw.write(`data: ${JSON.stringify(event)}\n\n`);
  };

  try {
    const stream = await app.stream(
      { messages },
      { configurable: { thread_id }, streamMode: "messages" },
    );

    for await (const [chunk, metadata] of stream) {
      const node = metadata?.langgraph_node ?? "unknown";
      if (chunk instanceof AIMessageChunk) {
        send({ type: "token", content: chunk.content, node });
      }
    }
    send({ type: "done" });
  } catch (err) {
    send({ type: "error", content: err instanceof Error ? err.message : String(err) });
  } finally {
    reply.raw.end();
  }
});

// Resume after interrupt
server.post("/:app/resume", async (req, reply) => {
  const { thread_id, decision } = req.body as { thread_id: string; decision: string };
  const result = await app.invoke(
    new Command({ resume: decision }),
    { configurable: { thread_id } },
  );
  return reply.send({ lastMessage: result.messages.at(-1)?.content });
});

// Inspect thread history (time-travel)
server.get("/:app/threads/:threadId/history", async (req, reply) => {
  const config = { configurable: { thread_id: req.params.threadId } };
  const states = [];
  for await (const snapshot of app.getStateHistory(config)) {
    states.push({ values: snapshot.values, next: snapshot.next, tasks: snapshot.tasks });
  }
  return reply.send({ history: states });
});
```

### Langfuse tracing — optional callback injection

Add observability without polluting the graph definition. Instantiate the handler once; pass it via `callbacks` per-invocation only when enabled.

Seen in: [`agentailor/fullstack-langgraph-nextjs-agent`](https://github.com/agentailor/fullstack-langgraph-nextjs-agent) — `src/services/agentService.ts`

```bash
npm i @langfuse/langchain
```

```ts
import { CallbackHandler } from "@langfuse/langchain";

// Only instantiate when tracing is explicitly enabled — avoids errors when credentials are absent
const langfuseHandler =
  process.env.LANGFUSE_ENABLED === "true" ? new CallbackHandler() : null;

const result = await agent.stream(inputs, {
  streamMode: ["updates"],
  configurable: { thread_id },
  ...(langfuseHandler ? { callbacks: [langfuseHandler] } : {}),
});
```

### useStream hook — React frontend for any LangGraph server

Connect a React app to any LangGraph deployment with one hook. Handles threads, streaming, interrupts, and generative UI.

Seen in: [`langchain-ai/agent-chat-ui`](https://github.com/langchain-ai/agent-chat-ui) — `src/providers/Stream.tsx`

```bash
npm i @langchain/langgraph-sdk
```

```tsx
import { useStream } from "@langchain/langgraph-sdk/react";
import { uiMessageReducer, isUIMessage } from "@langchain/langgraph-sdk/react-ui";
import type { Message, UIMessage, RemoveUIMessage } from "@langchain/langgraph-sdk";

// Define state + update types for full type-safety
type StateType = { messages: Message[]; ui?: UIMessage[] };
const useTypedStream = useStream<
  StateType,
  {
    UpdateType: {
      messages?: Message[] | Message;
      ui?: (UIMessage | RemoveUIMessage)[];
    };
    CustomEventType: UIMessage | RemoveUIMessage;
  }
>;

function Chat() {
  const stream = useTypedStream({
    apiUrl: process.env.NEXT_PUBLIC_API_URL ?? "http://localhost:2024",
    apiKey: apiKey ?? undefined,
    assistantId: "agent",
    threadId: threadId ?? null,
    fetchStateHistory: true,          // load prior messages on mount
    onCustomEvent: (event, { mutate }) => {
      // Merge generative UI events into state
      if (isUIMessage(event)) {
        mutate((prev) => ({ ...prev, ui: uiMessageReducer(prev.ui ?? [], event) }));
      }
    },
    onThreadId: (id) => setThreadId(id),  // URL-sync the thread
  });

  return (
    <div>
      {stream.values?.messages?.map((m) => <div key={m.id}>{m.content as string}</div>)}
      <button
        onClick={() => stream.submit({ messages: [{ role: "user", content: input }] })}
        disabled={stream.isLoading}
      >
        Send
      </button>
      {/* Resume after interrupt */}
      {stream.interrupts?.map((i) => (
        <button key={i.ns?.join("/")} onClick={() => stream.submit(null, { command: { resume: "yes" } })}>
          Approve
        </button>
      ))}
    </div>
  );
}
```

---

### `PostgresStore` — production cross-thread memory with vector search

`PostgresStore` is the persistent counterpart to `InMemoryStore` — it survives process restarts and is shared across threads. Import from `@langchain/langgraph-checkpoint-postgres/store` (separate sub-path from the checkpointer).

**Simple singleton with env-based fallback** (from [`includeHasan/polyrag`](https://github.com/includeHasan/polyrag)):

```ts
import { PostgresStore } from "@langchain/langgraph-checkpoint-postgres/store";
import { InMemoryStore } from "@langchain/langgraph-checkpoint";
import type { BaseStore } from "@langchain/langgraph-checkpoint";

let _store: BaseStore | undefined;
let _pgStore: PostgresStore | undefined;

export function getStore(): BaseStore {
  if (_store) return _store;
  if (process.env.NODE_ENV === "production") {
    _pgStore = PostgresStore.fromConnString(process.env.DATABASE_URL!);
    _store = _pgStore;
  } else {
    _store = new InMemoryStore();  // dev: non-persistent
  }
  return _store;
}

// One-time table creation — idempotent, safe to call on startup
export async function setupStore() {
  if (process.env.NODE_ENV !== "production") return;
  if (!_pgStore) getStore();
  await _pgStore!.setup();
}

// Graceful shutdown
export async function closeStore() {
  if (_pgStore) { await _pgStore.stop(); _pgStore = undefined; _store = undefined; }
}

// Typed helpers — namespace: ["user", userId, "preferences"]
export async function putUserPreference(userId: string, key: string, value: unknown) {
  await getStore().put(["user", userId, "preferences"], key, { value });
}
export async function getUserPreference<T>(userId: string, key: string): Promise<T | undefined> {
  const item = await getStore().get(["user", userId, "preferences"], key);
  return (item?.value as { value?: T })?.value;
}
```

**Ensure-once setup (singleton promise)** (from [`Pritam-25/spendix-ai`](https://github.com/Pritam-25/spendix-ai)):

```ts
const memoryStore = PostgresStore.fromConnString(process.env.DATABASE_URL!);
let storeReady: Promise<void> | null = null;

export function ensureMemoryStoreReady() {
  if (!storeReady) {
    storeReady = memoryStore.setup().catch((err) => { storeReady = null; throw err; });
  }
  return storeReady;
}
```

**With vector embeddings for semantic search** (from [`codersgyan/codersgpt-genai`](https://github.com/codersgyan/codersgpt-genai)):

```ts
import { OpenAIEmbeddings } from "@langchain/openai";
import { PostgresStore } from "@langchain/langgraph-checkpoint-postgres/store";

const store = PostgresStore.fromConnString(process.env.DATABASE_URL!, {
  index: {
    dims: 1536,               // embedding dimensions
    embed: new OpenAIEmbeddings({ model: "text-embedding-3-small" }),
  },
});
await store.setup();
// store.search(namespace, { query: "user's intent" }) now uses vector similarity
```

**With connection-pool options and custom schema** (from [`fancyboi999/Loomic`](https://github.com/fancyboi999/Loomic)):

```ts
const store = new PostgresStore({
  connectionOptions: {
    connectionString: process.env.DATABASE_URL!,
    max: 3,                         // keep pool small on serverless/edge DB
    idleTimeoutMillis: 30_000,
    connectionTimeoutMillis: 10_000,
    keepAlive: true,
    keepAliveInitialDelayMillis: 10_000,
  },
  schema: "langgraph",              // custom PG schema for namespace isolation
});
await store.setup();
```

---

### `Send` as a dispatcher — LLM writes `next` field, generic router dispatches

From [`langchain-ai/open-canvas`](https://github.com/langchain-ai/open-canvas) (5.5k stars, production chat+editor):

The router node returns `new Send(state.next, { ...state })` where `state.next` was set by the preceding routing node. This avoids a long `if/else` in the conditional edge and lets you add new destination nodes without touching the router.

```ts
// 1. The "generatePath" node sets state.next (via rule-based checks or an LLM tool call)
export async function generatePath(
  state: typeof OpenCanvasGraphAnnotation.State,
  config: LangGraphRunnableConfig
): Promise<OpenCanvasGraphReturnType> {
  if (state.highlightedCode) return { next: "updateArtifact" };
  if (state.highlightedText) return { next: "updateHighlightedText" };
  if (state.webSearchEnabled) return { next: "webSearch" };
  // ... more rule checks, then fall through to LLM routing
  const { route } = await dynamicDeterminePath({ state, config });
  return { next: route };  // route = "replyToGeneralInput" | "generateArtifact" | "rewriteArtifact"
}

// 2. A generic dispatcher reads state.next and sends
const routeNode = (state: typeof OpenCanvasGraphAnnotation.State) => {
  if (!state.next) throw new Error("'next' field not set.");
  return new Send(state.next, { ...state });  // forward entire state to target node
};

// 3. Graph wires generatePath → routeNode (conditional), rest via addConditionalEdges
const graph = new StateGraph(OpenCanvasGraphAnnotation)
  .addNode("generatePath", generatePath)
  .addEdge(START, "generatePath")
  .addNode("updateArtifact", updateArtifact)
  .addNode("generateArtifact", generateArtifact)
  // ... all destination nodes
  .addConditionalEdges("generatePath", routeNode, [
    "updateArtifact", "generateArtifact", "rewriteArtifact",
    "replyToGeneralInput", "webSearch", /* ... */
  ])
  .compile().withConfig({ runName: "open_canvas" });
```

LLM routing uses a **forced tool call** to get a typed route string:

```ts
import { traceable } from "langsmith/traceable";

async function dynamicDeterminePathFunc({ state, config }) {
  const model = await getModelFromConfig(config, { temperature: 0, isToolCalling: true });
  const schema = z.object({
    route: z.enum(["replyToGeneralInput", "rewriteArtifact"]).describe("Next node"),
  });
  const modelWithTool = model.bindTools(
    [{ name: "route_query", description: "Route the user query.", schema }],
    { tool_choice: "route_query" }  // force the model to call this tool
  );
  const result = await modelWithTool.invoke([{ role: "user", content: formattedPrompt }]);
  return result.tool_calls?.[0]?.args as z.infer<typeof schema>;
}

// Wrap with LangSmith tracing outside of the graph
export const dynamicDeterminePath = traceable(dynamicDeterminePathFunc, {
  name: "dynamic_determine_path",
});
```

---

### Dual message lists — user-visible vs model-context

From [`langchain-ai/open-canvas`](https://github.com/langchain-ai/open-canvas):

`messages` is the source of truth shown to the user; `_messages` is what the model actually sees (can be replaced with a summary). The sentinel on the reducer clears the internal list when a summary arrives.

```ts
const OC_SUMMARIZED_MESSAGE_KEY = "is_summary_message";

export const OpenCanvasGraphAnnotation = Annotation.Root({
  // User-visible history — never trimmed
  ...MessagesAnnotation.spec,

  // Model context — replaced wholesale when a summary message arrives
  _messages: Annotation<BaseMessage[], Messages>({
    reducer: (state, update) => {
      const latestMsg = Array.isArray(update) ? update[update.length - 1] : update;
      const isSummary = (latestMsg as any)?.additional_kwargs?.[OC_SUMMARIZED_MESSAGE_KEY] === true;

      if (isSummary) {
        // Clear the old list — summary replaces everything
        return messagesStateReducer([], update);
      }
      return messagesStateReducer(state, update);
    },
    default: () => [],
  }),
});
```

---

### Async fire-and-forget summarizer via SDK client + cross-thread state write-back

From [`langchain-ai/open-canvas`](https://github.com/langchain-ai/open-canvas):

When history is too long the main graph fires off a background summarizer run (without awaiting it), which then writes the summary back to the original thread using `client.threads.updateState`.

```ts
// Main graph node — fire and forget
import { Client } from "@langchain/langgraph-sdk";

async function summarizerDispatchNode(state, config: LangGraphRunnableConfig) {
  const client = new Client({ apiUrl: `http://localhost:${process.env.PORT}` });

  // Create a fresh thread for the summarizer run to avoid sharing checkpointer state
  const { thread_id: summaryThreadId } = await client.threads.create();

  // Submit to the "summarizer" assistant — does NOT await the result
  await client.runs.create(summaryThreadId, "summarizer", {
    input: {
      messages: state._messages,
      threadId: config.configurable?.thread_id,  // pass original thread ID for write-back
    },
  });

  return {};  // main graph continues immediately; summary arrives asynchronously
}

// Summarizer graph (separate deployment) — writes back to original thread
async function summarize(state: SummarizeState) {
  const summary = await model.invoke([
    ["system", SUMMARIZER_PROMPT],
    ["user", formatMessages(state.messages)],
  ]);

  const summaryMessage = new HumanMessage({
    id: uuidv4(),
    content: `Summary of prior conversation:\n${summary.content}`,
    additional_kwargs: { [OC_SUMMARIZED_MESSAGE_KEY]: true },  // sentinel for reducer
  });

  const client = new Client({ apiUrl: `http://localhost:${process.env.PORT}` });

  // Write summary back to the ORIGINAL thread — the reducer clears _messages on receipt
  await client.threads.updateState(state.threadId, {
    values: { _messages: [summaryMessage] },
  });
}
```

`graph.name = "Summarizer Graph"` — alternative to `.withConfig({ runName })` for naming deployed graphs (shown in LangGraph Studio and LangSmith traces):

```ts
const graph = builder.compile();
graph.name = "Summarizer Graph";  // mutable property; sets the display name
```

---

### `Send` — map-reduce fan-out over dynamic list

Fan out to N parallel node instances, each with isolated state, then reduce. The conditional edge returns `Send[]` instead of a string.

From [`elastic/kibana`](https://github.com/elastic/kibana) (production security assistant):

```ts
import { StateGraph, START, END, Send, Annotation } from "@langchain/langgraph";

const SelectIndexPatternAnnotation = Annotation.Root({
  shortlistedIndexPatterns: Annotation<string[]>({
    reducer: (cur, next) => next ?? cur,
    default: () => [],
  }),
  // map-reduce: each parallel branch writes into this record by indexPattern key
  indexPatternAnalysis: Annotation<Record<string, { containsRequiredData: boolean; context: string }>>({
    reducer: (cur, next) => ({ ...cur, ...next }),  // merge results from all branches
    default: () => ({}),
  }),
});

const graph = new StateGraph(SelectIndexPatternAnnotation)
  .addNode("fetch_patterns", fetchIndexPatterns({ esClient }))
  .addNode("shortlist",      shortlistIndexPatterns, { retryPolicy: { maxAttempts: 3 } })
  .addNode("analyze",        getAnalyzeIndexPattern({ analyzeIndexPatternGraph }),
    { retryPolicy: { maxAttempts: 3 }, subgraphs: [analyzeIndexPatternGraph] })
  .addNode("select",         getSelectIndexPattern(), { retryPolicy: { maxAttempts: 3 } })
  .addEdge(START, "fetch_patterns")
  .addEdge("fetch_patterns", "shortlist")
  .addConditionalEdges(
    "shortlist",
    (state) => {
      if (state.shortlistedIndexPatterns.length === 0) return END;
      // fan out: one Send per index pattern, each gets its own isolated state slice
      return state.shortlistedIndexPatterns.map(
        (indexPattern) => new Send("analyze", { input: { question: state.input?.question, indexPattern } })
      );
    },
    { analyze: "analyze", [END]: END }
  )
  .addEdge("analyze", "select")
  .addEdge("select", END)
  .compile();
```

From [`liam-hq/liam`](https://github.com/liam-hq/liam) — Send from START to distribute SQL test cases:

```ts
function continueToRequirements(state: QaAgentState) {
  return getUnprocessedRequirements(state).map(
    (testcaseData) =>
      new Send("testcaseGeneration", {
        currentTestcase: testcaseData,
        schemaData: state.schemaData,
        goal: state.analyzedRequirements.goal,
        messages: [],  // isolated message history per branch
      })
  );
}

graph.addConditionalEdges(START, continueToRequirements);
```

Key: the **reducer** on the accumulating field (`indexPatternAnalysis`) must merge results from all branches. Each branch writes a disjoint key so merging is safe. The `subgraphs` option in `addNode` tells Studio to visualise the nested graph inline.

---

### `retryPolicy` and `subgraphs` in `addNode`

From [`elastic/kibana`](https://github.com/elastic/kibana):

```ts
.addNode("analyze", analyzeNode, {
  retryPolicy: { maxAttempts: 3 },   // auto-retry on transient failures
  subgraphs: [analyzeIndexPatternGraph],  // register nested graph for Studio visualisation
})
```

`retryPolicy` catches exceptions thrown by the node function and re-runs up to `maxAttempts` times. Useful for LLM calls that occasionally fail with rate-limit or timeout errors.

---

### `isGraphInterrupt` and `getWriter` — production patterns

From [`n8n-io/n8n`](https://github.com/n8n-io/n8n) (AI workflow builder, production):

**`isGraphInterrupt`** — When you wrap a node in try/catch, you MUST re-throw graph interrupts or the checkpoint/resume cycle breaks:

```ts
import { isGraphInterrupt, getWriter } from "@langchain/langgraph";

async function subgraphNodeHandler(state: ParentState, config?: RunnableConfig) {
  try {
    return await execute(state, config);
  } catch (error) {
    if (isGraphInterrupt(error)) throw error;  // must not swallow interrupt signals
    // handle real errors here
    logger.error("subgraph failed", { error });
    return { messages: [new AIMessage("An error occurred.")] };
  }
}
```

**`getWriter`** — write streaming chunks from inside a node (bypasses the normal message-return path):

```ts
import { getWriter } from "@langchain/langgraph";

async function assistantNode(state: ParentState, config?: RunnableConfig) {
  const streamWriter = getWriter(config);   // returns writer | undefined

  const result = await assistantHandler.execute(query, userId, (chunk) => {
    streamWriter?.(chunk);   // push token chunks to the SSE stream in real-time
  });

  return { messages: [new AIMessage(result.responseText)] };
}
```

**Auto-message compaction** — n8n's graph includes a `compact_messages` node triggered when the conversation history exceeds a token threshold, preventing context overflow in long-running agents:

```ts
const AUTO_COMPACT_THRESHOLD = 100_000; // tokens

.addNode("check_state", (state) => {
  const tokenEstimate = estimateTokens(state.messages);
  if (tokenEstimate > AUTO_COMPACT_THRESHOLD) return { nextPhase: "auto_compact_messages" };
  if (hasDanglingToolCalls(state.messages)) return { nextPhase: "cleanup_dangling" };
  return { nextPhase: "continue" };
})
.addNode("compact_messages", async (state, config) => {
  const compacted = await summarizeOldMessages(state.messages, llm, config);
  return { messages: compacted };
})
.addConditionalEdges("compact_messages", (state) =>
  state.messages.length > 0 ? "check_state" : "responder"   // auto: loop back; manual: end
)
```

---

### `getCurrentTaskInput` — read graph state from inside a tool

Tools receive only their declared input schema. To access the broader graph state (e.g. a pipeline, cached data, or the active test case), call `getCurrentTaskInput<StateType>()` without passing state as a parameter.

From [`elastic/kibana`](https://github.com/elastic/kibana) and [`liam-hq/liam`](https://github.com/liam-hq/liam):

```ts
import { getCurrentTaskInput, Command } from "@langchain/langgraph";
import { DynamicStructuredTool } from "@langchain/core/tools";

export function fetchCurrentPipelineTool(): DynamicStructuredTool {
  return new DynamicStructuredTool({
    name: "fetch_pipeline",
    schema: z.object({ start_index: z.number().optional(), end_index: z.number().optional() }),
    func: async (input, _runManager, config) => {
      // Read the parent graph's state without it being passed as a tool argument
      const state = getCurrentTaskInput<AutomaticImportAgentStateType>();
      const pipeline = state.current_pipeline ?? {};
      // ... use pipeline data ...
    },
  });
}
```

---

### `Command` returned from a tool — write to graph state

Tools normally return a string or `ToolMessage`. Returning a `Command` lets a tool write directly to any graph state field, not just `messages`.

From [`liam-hq/liam`](https://github.com/liam-hq/liam) (`saveTestcaseTool`) and [`elastic/kibana`](https://github.com/elastic/kibana):

```ts
import { Command, getCurrentTaskInput } from "@langchain/langgraph";
import { ToolMessage } from "@langchain/core/messages";
import { dispatchCustomEvent } from "@langchain/core/callbacks/dispatch";
import { tool } from "@langchain/core/tools";

export const saveTestcaseTool = tool(
  async (input: { sql: string }, config: RunnableConfig): Promise<Command> => {
    const toolCallId = config.toolCall?.id;

    // Validate the SQL
    const isValid = await validateSqlSyntax(input.sql);
    if (!isValid) {
      const errMsg = new ToolMessage({ name: "saveTestcase", status: "error",
        content: "Invalid SQL", tool_call_id: toolCallId });

      // Stream the error message to the frontend immediately
      await dispatchCustomEvent("messages", errMsg);

      // Throw to trigger LangGraph retry mechanism
      throw new Error("Invalid SQL syntax");
    }

    // Read state to get contextual data
    const state = getCurrentTaskInput<typeof testcaseAnnotation.State>();
    const { testcaseId } = state.currentTestcase;

    const successMsg = new ToolMessage({ name: "saveTestcase", status: "success",
      content: `Saved SQL for test case ${testcaseId}`, tool_call_id: toolCallId });

    await dispatchCustomEvent("messages", successMsg);

    // Return Command to write to a non-messages state field
    return new Command({
      update: {
        generatedSqls: [{ testcaseId, sql: input.sql }],  // write to custom state field
        messages: [successMsg],
      },
    });
  },
  { name: "saveTestcase", description: "Save SQL for the current test case.", schema: z.object({ sql: z.string() }) }
);
```

---

### `interrupt()` inside a tool — structured clarifying questions

From [`n8n-io/n8n`](https://github.com/n8n-io/n8n) — the agent calls a tool, the tool suspends execution to ask the user structured questions, then resumes with answers:

```ts
import { interrupt } from "@langchain/langgraph";
import { tool } from "@langchain/core/tools";

export const submitQuestionsTool = tool(
  async (input: { introMessage?: string; questions: PlannerQuestion[] }) => {
    // Pause the graph here — the frontend receives the interrupt payload
    const resumeValue: unknown = interrupt({
      type: "questions",
      introMessage: input.introMessage,
      questions: input.questions,  // [{ id, question, type: "single"|"multi"|"text", options? }]
    });

    // Graph resumes here with the user's answers
    return formatAnswersForDiscovery(input.questions, resumeValue);
  },
  {
    name: "submit_questions",
    description: "Ask clarifying questions before proceeding. Max 5 questions.",
    schema: z.object({
      introMessage: z.string().optional(),
      questions: z.array(z.object({
        id: z.string(),
        question: z.string(),
        type: z.enum(["single", "multi", "text"]),
        options: z.array(z.string()).optional(),
      })).min(1).max(5),
    }),
  }
);
```

---

### Self-looping node — retry until condition met

From [`Ajitesh1405/knot`](https://github.com/Ajitesh1405/knot) — the `askEmail` node loops back to itself until all attendee emails are resolved:

```ts
.addNode("askEmail", async (s: SchedulerState) => {
  const provided = interrupt<NeedEmailInterrupt, string>({
    kind: "need_email",
    names: s.unresolved,  // tell the frontend which names still need email addresses
  });
  // Parse email addresses from the user's free-text response
  const emails = (provided ?? "").match(/[^\s@]+@[^\s@]+\.[^\s@]+/g) ?? [];
  const resolved = [...s.resolved];
  const stillMissing: string[] = [];
  s.unresolved.forEach((name, i) => {
    if (emails[i]) resolved.push({ name, email: emails[i] });
    else stillMissing.push(name);
  });
  return { resolved, unresolved: stillMissing };
})
.addConditionalEdges(
  "askEmail",
  (s) => (s.unresolved.length > 0 ? "ask" : "slot"),
  { ask: "askEmail", slot: "findSlot" }  // "ask" routes back to itself
)
```

---

### Custom reducers for production state management

From [`n8n-io/n8n`](https://github.com/n8n-io/n8n) — real-world reducer patterns beyond basic `messagesStateReducer`:

```ts
export const ParentGraphState = Annotation.Root({
  // Standard: last-write-wins (replace reducer — y ?? x)
  workflowJSON: Annotation<SimpleWorkflow>({ reducer: (x, y) => y ?? x, default: () => ({}) }),

  // Append-only audit log (concat reducer)
  coordinationLog: Annotation<CoordinationLogEntry[]>({
    reducer: (x, y) => x.concat(y),
    default: () => [],
  }),

  // Dedup union — accumulate approved domains across subgraph runs, no duplicates
  approvedDomains: Annotation<string[]>({
    reducer: (x, y) => [...new Set([...x, ...y])],
    default: () => [],
  }),

  // Per-turn counter that ALWAYS resets (ignore old value entirely)
  webFetchCount: Annotation<number>({
    reducer: (_x, y) => y,   // discard x — y is always authoritative
    default: () => 0,
  }),
});

// Reusable reducer: dedup-merge array by ID field
export function cachedTemplatesReducer(
  current: WorkflowMetadata[],
  update: WorkflowMetadata[] | undefined | null
): WorkflowMetadata[] {
  if (!update?.length) return current;
  const existingById = new Map(current.map((wf) => [wf.templateId, wf]));
  for (const wf of update) {
    if (!existingById.has(wf.templateId)) existingById.set(wf.templateId, wf);
  }
  return Array.from(existingById.values());
}

// Reusable reducer: skip empty updates
export function appendArrayReducer<T>(current: T[], update: T[] | undefined | null): T[] {
  return update?.length ? [...current, ...update] : current;
}
```

---

### Dynamic `ToolNode` built at runtime using state values

From [`elastic/kibana`](https://github.com/elastic/kibana) — the tool set varies per execution based on which index pattern was selected:

```ts
.addNode("tools", (state: typeof GenerateEsqlAnnotation.State) => {
  const { selectedIndexPattern } = state;
  if (!selectedIndexPattern) throw new Error("selectedIndexPattern is required");

  // ToolNode is constructed inside the node using runtime state — not at compile time
  const toolNode = new ToolNode([
    getInspectIndexMappingTool({ esClient, indexPattern: selectedIndexPattern }),
  ]);
  return toolNode.invoke(state);
})
```

---

### Self-correcting agent loop — error report feeds back to LLM

From [`elastic/kibana`](https://github.com/elastic/kibana) (`generate_esql` graph):

```ts
const graph = new StateGraph(GenerateEsqlAnnotation)
  .addNode("nl_to_esql_agent",    nlToEsqlAgentNode, { retryPolicy: { maxAttempts: 3 } })
  .addNode("tools",               toolsNode)
  .addNode("validate_esql",       validateEsqlInLastMessageNode)
  .addNode("build_success_report", buildSuccessReportNode)
  .addNode("build_error_report",   buildErrorReportNode)
  .addEdge(START, "select_index_pattern_graph")
  .addConditionalEdges("nl_to_esql_agent", nlToEsqlStepRouter, {
    validate_esql: "validate_esql",
    tools: "tools",
  })
  .addEdge("tools", "nl_to_esql_agent")
  // Validation failure loops back to the LLM to self-correct
  .addEdge("build_error_report", "nl_to_esql_agent")
  .addConditionalEdges("validate_esql", validateEsqlStepRouter, {
    build_success_report: "build_success_report",  // valid → done
    build_error_report:   "build_error_report",    // invalid → loop back
  })
  .addEdge("build_success_report", END)
  .compile();
```

The error report node constructs a message summarising what was wrong with the ES|QL, which the LLM reads on the next iteration to correct itself. The `retryPolicy` handles transient API failures at the node level independently.

---

### `getStateHistory` and `forkFromCheckpoint` — time-travel and branching

From [`g122622/synthos`](https://github.com/g122622/synthos) (production chat analysis system):

```ts
// Paginated checkpoint history — iterate with cursor
async function getStateHistory(params: {
  conversationId: string;
  limit: number;
  beforeCheckpointId?: string;
}) {
  const graph = workflow.compile({ checkpointer });

  const options: CheckpointListOptions = { limit: params.limit };
  if (params.beforeCheckpointId) {
    options.before = {
      configurable: {
        thread_id: params.conversationId,
        checkpoint_id: params.beforeCheckpointId,
      },
    };
  }

  const items = [];
  for await (const snapshot of graph.getStateHistory(
    { configurable: { thread_id: params.conversationId } },
    options
  )) {
    const checkpointId = (snapshot.config as any)?.configurable?.checkpoint_id;
    if (!checkpointId) continue;
    items.push({
      checkpointId,
      createdAt: snapshot.createdAt ? Date.parse(snapshot.createdAt) : Date.now(),
      next: snapshot.next,           // which nodes would run next from this point
      metadata: snapshot.metadata,
    });
  }

  return { items, nextCursor: items.at(-1)?.checkpointId };
}

// Fork: clone state at a specific checkpoint into a new thread
async function forkFromCheckpoint(params: {
  conversationId: string;
  checkpointId: string;
  newConversationId?: string;
}) {
  const graph = workflow.compile({ checkpointer });

  // 1. Read state at the source checkpoint
  const snapshot = await graph.getState({
    configurable: {
      thread_id: params.conversationId,
      checkpoint_id: params.checkpointId,
    },
  });

  // 2. Write that state into a brand-new thread as the "fork" node
  const newId = params.newConversationId ?? crypto.randomUUID();
  await graph.updateState(
    { configurable: { thread_id: newId } },
    snapshot.values,
    "fork"   // as_node name — labels the checkpoint in the new thread
  );

  return { conversationId: newId };
}
```

---

### Typed discriminated-union interrupt payloads and multi-step HITL loop

From [`Ajitesh1405/knot`](https://github.com/Ajitesh1405/knot) (NestJS email assistant with Telegram frontend):

```ts
// Typed interrupt payloads — discriminated by `kind`
type ChooseSenderInterrupt = { kind: "choose_sender"; recipient: string; candidates: SenderOption[] };
type ApproveInterrupt     = { kind: "approve";        draft: EmailDraft };
type ComposeInterrupt     = ChooseSenderInterrupt | ApproveInterrupt;

type ApproveDecision = { action: "approve" | "edit" | "replace" | "cancel"; correction?: string; body?: string };

// Multiple interrupt() calls in different nodes — each pause/resume independently
const chooseSender = async (state: ComposeState) => {
  const payload: ChooseSenderInterrupt = { kind: "choose_sender", recipient: state.recipient, candidates: options };
  const choice = interrupt<ChooseSenderInterrupt, { index: number }>(payload);
  return { chosen: state.candidates[choice.index] };
};

const approve = async (state: ComposeState) => {
  const decision = interrupt<ApproveInterrupt, ApproveDecision>({ kind: "approve", draft: state.draft! });
  return { decision };
};

// Graph loops back from approve → redraft → approve until user confirms
new StateGraph(ComposeState)
  .addNode("resolve", resolve)
  .addNode("chooseSender", chooseSender)
  .addNode("drafting", draft)
  .addNode("approve", approve)
  .addNode("redraft", redraft)       // AI revises draft on "edit" decision
  .addNode("replaceBody", replaceBody)  // verbatim replacement on "replace"
  .addNode("send", send)
  .addNode("cancelled", cancelled)
  .addEdge(START, "resolve")
  .addConditionalEdges("resolve",
    (s) => s.status === "no_match" ? "end" : s.candidates?.length > 1 ? "choose" : "draft",
    { choose: "chooseSender", draft: "drafting", end: END })
  .addEdge("chooseSender", "drafting")
  .addEdge("drafting", "approve")
  .addConditionalEdges("approve",
    (s) => s.decision?.action === "approve" ? "send"
         : s.decision?.action === "edit"    ? "redraft"
         : s.decision?.action === "replace" ? "replace"
         : "cancel",
    { send: "send", redraft: "redraft", replace: "replaceBody", cancel: "cancelled" })
  .addEdge("redraft", "approve")     // loop: back to approval gate
  .addEdge("replaceBody", "approve") // loop: back to approval gate
  .addEdge("send", END)
  .addEdge("cancelled", END)
  .compile({ checkpointer });
```

---

### Max-rounds protection node

From [`g122622/synthos`](https://github.com/g122622/synthos) — cap runaway tool loops without throwing:

```ts
// Per-run counters in graph state (reset by passing initial values at invoke time)
const AgentState = Annotation.Root({
  messages:      Annotation<BaseMessage[]>({ reducer: messagesStateReducer, default: () => [] }),
  maxToolRounds: Annotation<number>,
  runToolRounds: Annotation<number>,   // incremented by llmCall node each iteration
});

const shouldContinue = (state: AgentState): "tools" | "maxRounds" | typeof END => {
  const last = state.messages.at(-1);
  if (!last || !AIMessage.isInstance(last)) return END;
  if (state.runToolRounds >= state.maxToolRounds) return "maxRounds";  // cap
  if ((last as AIMessage).tool_calls?.length) return "tools";
  return END;
};

const maxRoundsNode = async (): Promise<Partial<AgentState>> => ({
  messages: [new AIMessage("Max tool rounds reached. Please simplify the request.")],
});

const graph = new StateGraph(AgentState)
  .addNode("llmCall", llmCall)
  .addNode("tools", toolsNode)
  .addNode("maxRounds", maxRoundsNode)
  .addEdge(START, "llmCall")
  .addConditionalEdges("llmCall", shouldContinue, ["tools", "maxRounds", END])
  .addEdge("tools", "llmCall")
  .addEdge("maxRounds", END)
  .compile({ checkpointer });

// Reset per-run counters at invoke time — not in graph state defaults
await graph.invoke({ messages: [new HumanMessage(userMessage)], maxToolRounds: 5, runToolRounds: 0 },
  { configurable: { thread_id: conversationId } });
```

---

## Open-source references

| Project | Stars | What it demonstrates |
|---|---|---|
| [`langchain-ai/agents-from-scratch-ts`](https://github.com/langchain-ai/agents-from-scratch-ts) | — | Zod-based state with `.langgraph.reducer()`, nested subgraph as a node, `HumanInterrupt` config, `Command` named routing map |
| [`agentailor/fullstack-langgraph-nextjs-agent`](https://github.com/agentailor/fullstack-langgraph-nextjs-agent) | 118 | Tool approval node, `interrupt()` in tools, PostgresSaver, MCP dynamic loading, Langfuse tracing, SSE to React |
| [`ac12644/langgraph-starter-kit`](https://github.com/ac12644/langgraph-starter-kit) | — | Supervisor + Swarm + RAG + HITL; lazy Postgres checkpointer; Fastify SSE server; time-travel history endpoint |
| [`langchain-ai/fullstack-chat-server`](https://github.com/langchain-ai/fullstack-chat-server) | — | `ConfigurationSchema` for runtime model selection, `Auth`+`HTTPException` JWT auth via Supabase, Stripe credits |
| [`langchain-ai/langgraphjs-gen-ui-examples`](https://github.com/langchain-ai/langgraphjs-gen-ui-examples) | — | `typedUi().push()` generative UI from nodes; stockbroker + trip planner + email agent examples |
| [`langchain-ai/agent-chat-ui`](https://github.com/langchain-ai/agent-chat-ui) | 2.9k | React `useStream` hook with interrupt resume, generative UI via `uiMessageReducer`, thread URL sync |
| [`langchain-ai/deepagentsjs`](https://github.com/langchain-ai/deepagentsjs) | 1.4k | Batteries-included agent harness; `createDeepAgent()` returns a compiled LangGraph; powers open-swe |
| [`mayooear/ai-pdf-chatbot-langchain`](https://github.com/mayooear/ai-pdf-chatbot-langchain) | 16.5k | Split ingestion + retrieval graphs, Next.js frontend; archived but clear reference architecture |
| [`proactive-agent/langgraphics`](https://github.com/proactive-agent/langgraphics) | 124 | Developer tool: `watch(compiledGraph)` opens a live browser visualisation of graph execution |
| [`elastic/kibana`](https://github.com/elastic/kibana/tree/main/x-pack/solutions/security/plugins/security_solution/server/assistant/tools/esql/graphs) | 21k | Production `Send` map-reduce for parallel index-pattern analysis; `retryPolicy`, `subgraphs` in `addNode` |
| [`n8n-io/n8n`](https://github.com/n8n-io/n8n/tree/master/packages/%40n8n/ai-workflow-builder.ee/src) | 100k+ | Production multi-agent with subgraphs; `isGraphInterrupt`, `getWriter`, auto-compaction, per-stage LLMs |
| [`liam-hq/liam`](https://github.com/liam-hq/liam/tree/main/frontend/internal-packages/agent/src/qa-agent) | — | `Send` from START to fan-out parallel test-case generation with isolated state per branch |
| [`yokingma/SearChat`](https://github.com/yokingma/SearChat/tree/main/packages/deepresearch) | — | Deep research graph: `Send` for parallel queries, `ConfigurationSchema`, node-scoped `input` state |
| [`g122622/synthos`](https://github.com/g122622/synthos) | 29 | Production `getStateHistory` with cursor pagination; `forkFromCheckpoint` via `getState`+`updateState`; max-rounds protection node |
| [`Ajitesh1405/knot`](https://github.com/Ajitesh1405/knot) | — | Typed discriminated-union interrupt payloads; multi-step approve→redraft→approve loop; NestJS integration |
| [`langchain-ai/open-canvas`](https://github.com/langchain-ai/open-canvas) | 5.5k | Real production app (chat+editor). `Send(state.next, state)` dispatcher; dual message lists (`messages` vs `_messages`); per-assistant `BaseStore` reflections; `withConfig({ runName })` for tracing |
| [`DimiMikadze/orca`](https://github.com/DimiMikadze/orca) | 1.3k | LinkedIn profile analysis agent. `createAgent` with `tool()` helpers, tool-result caching pattern, structured Zod output schema |
| [`includeHasan/polyrag`](https://github.com/includeHasan/polyrag) | — | Full `PostgresStore` abstraction: env-based prod/dev fallback, typed preference helpers, `store.stop()` on shutdown |
| [`codersgyan/codersgpt-genai`](https://github.com/codersgyan/codersgpt-genai) | 4 | `PostgresStore` with vector embeddings (`index: { dims, embed }`) for semantic memory search |
| [`fancyboi999/Loomic`](https://github.com/fancyboi999/Loomic) | — | `PostgresStore` with connection pool options and custom `schema:` for namespace isolation |
| [`xpert-ai/xpert`](https://github.com/xpert-ai/xpert) | 412 | Enterprise AI platform. Custom `ToolNode` with `setContextVariable`; `isCommand`+`isGraphInterrupt` in tool error handler; `dispatchCustomEvent` for tool errors; Plan Mode middleware |
