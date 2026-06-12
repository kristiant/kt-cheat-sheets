# Agentic Patterns (LangGraph.js)

**What it is:** Patterns for orchestrating LLM calls, tools, and sub-agents into reliable systems — sequential chains, parallel fan-out, routing, agent loops, and multi-agent teams.

**Why people use it:** A single LLM call can't plan, use tools, retry, or coordinate. A harness adds control flow, state, persistence, and human checkpoints so the system is debuggable and production-safe.

**Typically used for:** Tool-using agents, research/coding assistants, multi-step workflows, and any task needing loops, memory, or human approval.

> Anthropic's taxonomy ([Building Effective Agents](https://www.anthropic.com/research/building-effective-agents)) splits these into **workflows** (predetermined control flow) and **agents** (the LLM drives control flow). Prefer the simplest that works; reach for agents only when you need model-driven flexibility.

---

## LCEL vs LangGraph — pick the primitive

**LCEL** (LangChain Expression Language) = the `.pipe()` composition style; everything composed is a **Runnable** sharing `.invoke()`/`.batch()`/`.stream()`.

| Need | Use |
|---|---|
| Stateless, acyclic flow; simple parallel/branch | **LCEL** (`RunnableParallel`, `RunnableSequence`, `.batch()`) |
| Loops, persistent state, human-in-the-loop, multi-agent | **LangGraph** (`StateGraph` + checkpointer) |

```ts
// LCEL: fan out independent calls, await all
import { RunnableParallel } from "@langchain/core/runnables";

const parallel = RunnableParallel.from({ summary: summarize, keywords: extract });
const { summary, keywords } = await parallel.invoke(doc);

// LCEL: same chain over many inputs, concurrently
const results = await chain.batch(inputs, { maxConcurrency: 5 });
```

Everything below uses LangGraph (`npm i @langchain/langgraph @langchain/core`).

---

## Core model

State is a typed object; nodes return partial updates; **reducers** decide how updates merge (critical once nodes run in parallel).

```ts
import { Annotation, StateGraph, START, END } from "@langchain/langgraph";

const State = Annotation.Root({
  topic: Annotation<string>,
  drafts: Annotation<string[]>({
    reducer: (a, b) => a.concat(b),   // concurrent writes append, not overwrite
    default: () => [],
  }),
});

const graph = new StateGraph(State)
  .addNode("draft", async (s) => ({ drafts: [await llm.invoke(s.topic)] }))
  .addEdge(START, "draft")
  .addEdge("draft", END)
  .compile();
```

---

## Workflow patterns

### Prompt chaining

Sequential steps, each acting on the last. Optional gate between to fail fast.

```ts
new StateGraph(State)
  .addNode("outline", outlineNode)
  .addNode("write", writeNode)
  .addEdge(START, "outline")
  .addEdge("outline", "write")     // add a conditional gate here to bail on bad outlines
  .addEdge("write", END);
```

### Routing

Classify the input, dispatch to a specialised node.

```ts
graph.addConditionalEdges("classify", (s) => s.category, {
  billing:   "billingNode",
  technical: "techNode",
  other:     "fallbackNode",
});
```

### Parallelization — sectioning (map-reduce)

Fan out independent subtasks with the **`Send`** API, fan back in via a reducer. Each `Send` runs the node with its own state.

```ts
import { Send } from "@langchain/langgraph";

// conditional edge returns one Send per subject → N parallel runs of "writeSection"
graph.addConditionalEdges(
  "plan",
  (s) => s.sections.map((sec) => new Send("writeSection", { section: sec })),
  ["writeSection"],
);
// writeSection returns { drafts: [text] }; the reducer concatenates all of them
```

### Parallelization — voting

Run the same task N times, aggregate (majority / best-of).

```ts
graph.addConditionalEdges(
  "start",
  () => Array.from({ length: 3 }, () => new Send("classify", state)),
  ["classify"],
);
// then an "aggregate" node takes a majority vote over the 3 results
```

### Orchestrator-worker

An orchestrator decides subtasks **at runtime** (vs fixed sections), spawns a worker per subtask, synthesises. Same `Send` mechanism, but the fan-out list is LLM-generated.

```ts
async function orchestrate(s) {
  const plan = await llm.withStructuredOutput(PlanSchema).invoke(s.task);
  return { subtasks: plan.subtasks };
}
graph.addConditionalEdges("orchestrate",
  (s) => s.subtasks.map((t) => new Send("worker", { task: t })),
  ["worker"]);
```

### Evaluator-optimizer

Generate → evaluate → loop until a quality bar (or max iterations). A conditional edge routes back to the generator.

```ts
graph
  .addNode("generate", generateNode)
  .addNode("evaluate", evaluateNode)   // returns { passed: boolean, feedback }
  .addEdge("generate", "evaluate")
  .addConditionalEdges("evaluate",
    (s) => (s.passed || s.iter >= 3 ? "done" : "generate"),  // cap iterations
    { generate: "generate", done: END });
```

---

## Agent patterns

### ReAct agent (tool-calling loop)

The core agent: reason → call tool → observe → repeat until done. Use the prebuilt.

```ts
import { createReactAgent } from "@langchain/langgraph/prebuilt";
import { tool } from "@langchain/core/tools";
import { z } from "zod";

const getWeather = tool(async ({ city }) => `${city}: 22°C`, {
  name: "get_weather",
  description: "Current weather for a city",
  schema: z.object({ city: z.string() }),
});

const agent = createReactAgent({ llm, tools: [getWeather] });
const out = await agent.invoke({ messages: [{ role: "user", content: "weather in Perth?" }] });
```

### Multi-agent — supervisor

A supervisor routes work to specialist agents and collects results (tool-based handoff).

```ts
import { createSupervisor } from "@langchain/langgraph-supervisor";

const research = createReactAgent({ llm, tools: [search], name: "researcher" });
const maths    = createReactAgent({ llm, tools: [calculator], name: "maths" });

const app = createSupervisor({
  agents: [research, maths],
  llm,
  prompt: "Route research to researcher, calculations to maths.",
}).compile();
```

### Multi-agent — swarm / handoffs

Peer agents pass control to each other (no central router) via handoff tools. Use [`@langchain/langgraph-swarm`](https://github.com/langchain-ai/langgraphjs) when control should flow agent-to-agent rather than through a supervisor.

> **Pick:** supervisor for clear delegation/hierarchy; swarm for fluid peer handoffs. Start single-agent — add agents only when one agent's tool list gets unwieldy or domains genuinely separate.

---

## Harness layer

### State & reducers (concurrency)

The reducer is what makes parallel fan-out safe. Without one, two nodes writing the same key in the same step **conflict**. Append-reducers (`(a, b) => a.concat(b)`) or last-write reducers resolve it deterministically. For message state, LangGraph ships `MessagesAnnotation`.

### Persistence / checkpointing

A checkpointer snapshots state after every step → resumable threads, memory across turns, time-travel.

```ts
import { MemorySaver } from "@langchain/langgraph";

const app = graph.compile({ checkpointer: new MemorySaver() });   // prod: Postgres/SQLite saver
const config = { configurable: { thread_id: "user-42" } };

await app.invoke({ topic: "x" }, config);   // each thread_id is an isolated conversation
await app.invoke({ topic: "y" }, config);   // resumes with prior state
```

### Human-in-the-loop

Pause mid-graph for approval/edits, then resume from the exact checkpoint. Needs a checkpointer.

```ts
import { interrupt, Command } from "@langchain/langgraph";

function approveTool(s) {
  const decision = interrupt({ pendingAction: s.action });   // graph pauses here
  return { approved: decision };
}
// resume by re-invoking with the human's answer:
await app.invoke(new Command({ resume: true }), config);
```

Static alternative — pause before specific nodes at compile time:

```ts
graph.compile({ checkpointer, interruptBefore: ["tools"] });
```

### Streaming

Stream step progress *and* tokens. Pick the mode to fit the UI.

```ts
for await (const chunk of await app.stream(input, { streamMode: "updates" })) {
  console.log(chunk);   // per-node state deltas
}
// "values" full state each step · "updates" node deltas · "messages" LLM tokens · "custom" your events
```

### Durability & limits

```ts
await app.invoke(input, { recursionLimit: 25 });   // runaway-loop guard (default 25)
```

- **`recursionLimit`** caps total node steps — the backstop against an agent that never stops.
- **Retries** — set `retryPolicy` on a node for flaky tools/APIs.
- **Errors** — throw inside a node to halt; catch and write an error field to route to a recovery node instead.

### Subgraphs

Compose a compiled graph as a node in a parent graph — how hierarchical/team agents are built.

```ts
parent.addNode("research_team", researchSubgraph);   // a compiled StateGraph
```

### Observability

```bash
export LANGSMITH_TRACING=true
export LANGSMITH_API_KEY=ls-...
```

Traces every node, tool call, and token — essential for debugging multi-step runs.

---

## Practical recipes

**Tool agent with memory (most common starting point):**
```ts
const agent = createReactAgent({ llm, tools, checkpointer: new MemorySaver() });
await agent.invoke({ messages }, { configurable: { thread_id: "1" } });
```

**Fan out N subtasks and aggregate (map-reduce):**
```ts
graph.addConditionalEdges("plan",
  (s) => s.items.map((i) => new Send("work", { item: i })),
  ["work"]);   // "work" appends to a reducer-merged array; "reduce" node aggregates
```

**Approval gate before any tool runs:**
```ts
const app = graph.compile({ checkpointer: new MemorySaver(), interruptBefore: ["tools"] });
// inspect app.getState(config).next, then resume: await app.invoke(null, config);
```

**Self-correcting loop with a cap:**
```ts
graph.addConditionalEdges("evaluate",
  (s) => (s.passed || s.iter >= 3 ? END : "generate"),
  { generate: "generate", [END]: END });
```

**Stream tokens to a UI:**
```ts
for await (const [msg] of await app.stream(input, { streamMode: "messages" })) {
  process.stdout.write(msg.content);
}
```

---

## Failure modes & fixes

| Symptom | Cause | Fix |
|---|---|---|
| Parallel nodes overwrite each other's state | no reducer on the shared key | add an append/merge reducer |
| `GraphRecursionError` | agent loops forever | raise `recursionLimit`, or add a termination condition |
| State lost between turns | no checkpointer / reused `thread_id` | compile with a checkpointer, unique `thread_id` per conversation |
| Can't resume after human input | interrupt without persistence | interrupts require a checkpointer |
| Multi-agent ping-pongs / never finishes | unclear handoff rules | tighten supervisor prompt; cap steps |
| Tool call fails the whole run | unhandled node error | `retryPolicy` for flaky calls; route errors to a recovery node |

## Tips

- Start with the **simplest** pattern (single LLM call → chain → router → agent). Add structure only when it fails.
- A reducer on every concurrently-written key — non-negotiable for parallel graphs.
- Checkpointer + unique `thread_id` is what gives you memory, resume, and human-in-the-loop. It's the harness, not an add-on.
- Cap loops everywhere — `recursionLimit`, iteration counters, supervisor step limits.
- Prefer `createReactAgent` over hand-rolling the tool loop unless you need custom control flow.
- Trace with LangSmith before you debug by `console.log` — multi-step state is hard to reason about blind.
