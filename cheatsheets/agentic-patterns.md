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

## Reliability

Wrapping model/tool calls so a non-deterministic, sometimes-failing dependency doesn't sink the run.

**Retries with backoff** — re-attempt transient failures with exponential, jittered delays. Naïve immediate retries make a `429` storm worse.

```ts
const robust = model.withRetry({ stopAfterAttempt: 4 });   // built-in exponential backoff
```

**Model fallback chain** — on error/timeout, fall through to an alternative model/provider; survive an outage or rate-limit.

```ts
const llm = primary.withFallbacks({ fallbacks: [secondary, cheapLocal] });
```

**Timeout** — bound how long you wait; a hung call shouldn't hang the request.

```ts
await model.invoke(prompt, { signal: AbortSignal.timeout(10_000) });
```

- **Circuit breaker** — after repeated failures, stop calling a dependency for a cooldown (fail fast). No LangChain primitive — wrap it (`opossum`).
- **Idempotency** — make handlers safe to re-run via a dedup key; at-least-once queues and retries *will* double-invoke. See [aws-sqs.md](aws-sqs.md), [aws-lambda.md](aws-lambda.md).
- **Dead-letter queue** — park repeatedly-failing inputs instead of retrying forever, so one poison message doesn't block the pipeline.
- **Graceful degradation** — return a reduced-but-useful result on failure (cached answer, smaller model, "can't do that now") rather than an error.

### Concurrency control

- **Rate limiting / throttling** — keep request rate under provider quotas (RPM/TPM); the #1 cause of `429`s. Needs client-side limiting *and* backoff.
- **Concurrency limit** — cap simultaneous calls. In LangChain it's `maxConcurrency` on `.batch()`; for raw calls, a semaphore (`p-limit`).
- **Backpressure** — slow producers when consumers fall behind (queue depth as signal), so a burst doesn't overwhelm a downstream model/DB.
- **Hedging** — fire a duplicate after a delay, take the first to return; cuts p99 latency at the cost of extra spend.

```ts
const hedge = (fn, ms) => Promise.race([fn(), delay(ms).then(fn)]);
```

---

## Cost & performance

**Prompt caching** — provider caches the KV for a stable prompt prefix, skipping re-processing on repeat calls. Put stable content (system prompt, tool defs, docs) first, variable content (the user turn) last.

```text
[ system ][ tool defs ][ retrieved docs ]   ← stable, cacheable prefix
[ user turn ]                                ← varies, goes last
```

- **Semantic caching** — cache keyed by embedding similarity, not exact string; "what's your refund policy?" and "how do refunds work?" hit the same entry.
- **Batching** — group inputs into one `.batch()` call for throughput when latency isn't critical (offline/async).
- **Token budgeting** — track and cap tokens across prompt + retrieved context + history; budget so docs don't crowd out the question.
- **Context compaction** — summarise or trim old conversation turns into a running summary instead of dropping them when the window fills.

---

## Quality & control

**Structured output** — force responses into a schema and get a typed object back; turns "usually valid JSON" into a guarantee. LangChain + Zod binds and parses in one step:

```ts
import { z } from "zod";
const Out = z.object({ category: z.enum(["refund", "billing", "other"]) });

const classifier = model.withStructuredOutput(Out);
const { category } = await classifier.invoke(input);   // typed + schema-valid — see zod.md
```

- **Guardrails** — validate input/output, check schemas, filter content around the model; make the *system* dependable despite a non-deterministic core.
- **Grounding / citations** — answer only from supplied context and cite sources; the main lever against hallucination and it makes answers verifiable. See [rag.md](rag.md).
- **LLM-as-judge** — score one model's output with another against criteria; scalable eval and runtime quality gates where exact-match can't work.
- **Eval harness** — a dataset + metrics in CI to catch regressions; prompts and models change silently otherwise. See [rag.md](rag.md) (Ragas).
- **Hallucination detection** — flag unsupported claims (low logprobs, faithfulness check, self-consistency) before they reach users.

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
| `429` rate-limit errors under load | unbounded fan-out / no throttle | `maxConcurrency` on `.batch()` + client-side rate limit + backoff |
| Duplicate side effects (double charge) | retries / at-least-once delivery | idempotency key on the handler |
| Invalid JSON from the model | trusting raw output | `withStructuredOutput(ZodSchema)`; validate at the boundary |

## Tips

- Start with the **simplest** pattern (single LLM call → chain → router → agent). Add structure only when it fails.
- A reducer on every concurrently-written key — required for parallel graphs.
- Checkpointer + unique `thread_id` is what gives you memory, resume, and human-in-the-loop. It's the harness, not an add-on.
- Cap loops everywhere — `recursionLimit`, iteration counters, supervisor step limits.
- Prefer `createReactAgent` over hand-rolling the tool loop unless you need custom control flow.
- Bound everything parallel — `maxConcurrency` + rate limiting prevents the `429` storms that sink LLM apps.
- Retries need backoff + jitter; `.withRetry()` and `.withFallbacks()` handle the common cases.
- Validate model output with a Zod schema at every boundary; never trust shape.
- Trace with LangSmith before you debug by `console.log` — multi-step state is hard to reason about blind.
