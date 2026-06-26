# Building an Agentic Product on an Agent SDK

**What it is:** The architecture for turning a low-level agent/LLM SDK (Vercel AI SDK) into a *product-grade* agent runtime — a declarative authoring layer, an execution engine that owns the loop, and the wiring for suspend/resume, sub-agents, memory, and observability.

**Why people use it:** The AI SDK gives you `generateText`/`streamText` and a basic tool loop. A product needs more: human-in-the-loop, durable checkpoints, concurrency control, multi-provider models, guardrails, evals, and telemetry. This is how you layer those on without leaking the SDK everywhere.

**Typically used for:** Agent platforms, internal agent runtimes, anything that runs *other people's* agents and must be controllable, resumable, and observable.

> Primarily grounded in [n8n](https://github.com/n8n-io/n8n)'s `@n8n/agents` (built on the [Vercel AI SDK](https://sdk.vercel.ai)); file paths below are relative to that package. Corroborated against other open-source ADKs — [Mastra](https://github.com/mastra-ai/mastra), [LangGraph](https://github.com/langchain-ai/langgraphjs), and LangChain [deepagents](https://github.com/langchain-ai/deepagentsjs) — in [Patterns across frameworks](#patterns-across-frameworks) so the patterns aren't one library's idiosyncrasies.

---

## The two-layer split: SDK vs runtime

The core structural split: separate the **declarative authoring layer** (`sdk/`) from the **execution engine** (`runtime/`). The AI SDK is touched *only* in the runtime.

```
sdk/        fluent builders — Agent, Tool, Memory, Guardrail, Eval   (public)
runtime/    AgentRuntime, tool-adapter, model-factory, message-list  (internal, never exported)
types/      public contracts (BuiltAgent, BuiltTool, StreamChunk, …)
```

- `sdk/` is what users compose. It produces plain data (`BuiltAgent`, `BuiltTool`) — no execution.
- `runtime/` consumes that data and drives the AI SDK. It is *not* exported.
- The boundary between them is a set of `Built*` types in `types/`.

Why it matters: you can change the engine (loop strategy, provider, streaming) without touching user code, and users can't reach into execution internals.

---

## Builder pattern with lazy build

Every primitive is a fluent builder; **user code never calls `.build()`**. The consuming method builds internally.

```ts
const tool = new Tool('web-search')
  .description('Search the web')
  .input(z.object({ query: z.string() }))
  .output(z.object({ results: z.array(z.string()) }))
  .handler(async ({ query }) => ({ results: [`found: ${query}`] }));

const agent = new Agent('researcher')
  .model('anthropic/claude-sonnet-4-5')
  .instructions('You are a research assistant.')
  .tool(tool)                       // agent.tool() calls tool.build() internally
  .memory(new Memory());

await agent.generate('Find RAG papers');   // lazy-builds the agent on first call
```

Wiring (`sdk/agent.ts`):
- `build()` is **`protected`** — kept out of the public surface.
- `generate()`/`stream()` call `ensureBuilt()` which builds once and caches.
- Builders carry typed schemas (Zod) so `.handler()` is fully inferred from `.input()`/`.output()`.

This keeps the public API a pure description and defers all resolution (model, credentials, tools) to one controlled point.

---

## Engine injection via subclassing

The product layer extends the SDK builder and overrides `protected build()` to inject infrastructure *before* `super.build()`. This is how platform concerns (credential resolution, durable checkpoints) attach without polluting the SDK.

```ts
class EngineAgent extends Agent {
  protected build() {
    this.checkpoint(this.durableStore);         // swap in a real CheckpointStore
    if (this.declaredCredential) {
      this.resolvedApiKey = resolve(this.declaredCredential);   // .credential('anthropic') → key
    }
    return super.build();
  }
}
```

Users write `.credential('anthropic')` — a *name*, never a key. The engine resolves it at build time and injects it into the model config. Same hook point for any infra concern.

---

## Own the loop — wrap the AI SDK, don't delegate to it

The AI SDK can run a tool loop itself (via `maxSteps`/auto-executed tools). This runtime takes that over: tools are handed to the SDK *without* an `execute` function, so the model proposes calls but the runtime executes them.

```ts
// tool-adapter.ts — toAiSdkTools(): note there is NO `execute`
result[t.name] = ai.tool({
  description: t.description,
  inputSchema: t.inputSchema,        // Zod, or ai.jsonSchema(...) for MCP tools
  providerOptions: t.providerOptions,
});
```

The runtime then runs its own loop (`agent-runtime.ts`, `MAX_LOOP_ITERATIONS = 20`):

```
emit TurnStart
  → generateText/streamText with list.forLlm(...)
  → addResponse(assistant + tool calls)
  → execute tool calls in batches of toolCallConcurrency (Promise.allSettled)
  → handle any suspension
  → repeat until finish or max iterations
```

Owning the loop is what enables per-batch concurrency, suspend mid-batch, HITL gates, abort checks between batches, and custom persistence — none of which are reachable when the SDK drives the loop.

> `toolCallConcurrency` defaults to `1` (sequential); `Infinity` runs all executable calls in one turn in parallel. Each batch is `Promise.allSettled` so one failure doesn't sink the others.

---

## Model factory — provider-agnostic model strings

Models are addressed as `"provider/model-name"` strings, resolved through a registry of lazily-required AI SDK provider packages, with per-provider Zod-validated credentials.

```ts
// model-factory.ts (condensed)
const LANGUAGE_PROVIDERS = {
  anthropic: { build: (creds, model, fetch) => {
    const { createAnthropic } = require('@ai-sdk/anthropic');
    return createAnthropic({ ...creds, fetch })(model);
  }},
  openai:    { build: (creds, model, fetch) => { /* @ai-sdk/openai */ } },
  // google, xai, bedrock, …
};

export function createModel(config) {
  if (isLanguageModel(config)) return config;        // pass through a pre-built model
  const slash = rawId.indexOf('/');
  const provider = rawId.slice(0, slash);            // "anthropic"
  const modelName = rawId.slice(slash + 1);          // "claude-sonnet-4-5"
  const creds = PROVIDER_CREDENTIAL_SCHEMAS[provider].parse(credFields);   // Zod-validated
  return LANGUAGE_PROVIDERS[provider].build(creds, modelName, getProxyFetch());
}
```

Key points:
- **`require()` per provider, not top-level imports** — only the providers actually used get loaded; the rest stay out of the bundle.
- **Credentials validated per provider** with a Zod schema before the SDK ever sees them — clear errors instead of opaque SDK failures.
- **Injectable `fetch`** — a proxy-aware fetch is threaded into every provider (Node's `fetch` ignores `HTTP_PROXY`), so test/proxy setups work without per-call changes.

---

## Suspend / resume + HITL via branded results

The hardest product requirement is pausing a run for human input (or a long external action) and continuing later. The mechanism: a tool returns a **branded** suspend object; the runtime detects it, checkpoints, and returns control.

```ts
// a tool that pauses for confirmation
new Tool('write-file')
  .input(z.object({ path: z.string(), content: z.string() }))
  .suspend(z.object({ message: z.string() }))     // payload shown to the caller
  .resume(z.object({ approved: z.boolean() }))    // payload expected back
  .handler(async ({ path, content }, ctx) => {
    if (!ctx.resumeData) return await ctx.suspend({ message: `Write to ${path}?` });
    if (!ctx.resumeData.approved) return { written: false };
    return { written: true };
  });
```

Wiring:
- `ctx.suspend(payload)` returns `{ [SUSPEND_BRAND]: true, payload }` — a sentinel the runtime recognises (`isSuspendedToolResult`), not a thrown error.
- On detection the runtime persists `pendingToolCalls` + the message list to a **`CheckpointStore`** (`RunStateManager`), keyed by `runId`, and returns `pendingSuspend` (generate) or a `tool-call-suspended` chunk (stream).
- `resume(method, data, { runId, toolCallId })` loads the checkpoint and re-enters the loop. `complete(runId)` deletes it when the run finishes.

**HITL is the same machinery.** `approve()`/`deny()` just call `resume()` with `{ approved: true|false }`. Any tool becomes interruptible by wrapping it to inject approval schemas:

```ts
// wrapToolForApproval(): if approval is needed, suspend before running the real handler
suspendSchema: APPROVAL_SUSPEND_SCHEMA,   // { type:'approval', toolName, args }
resumeSchema:  APPROVAL_RESUME_SCHEMA,    // { approved: boolean }
```

The `CheckpointStore` is an interface (default in-memory, pluggable for Postgres/Redis) — durability is a swap, not a rewrite.

---

## Message list with provenance

One turn's messages live in a single append-only array plus **three Sets** (history / input / response) tagged by stable message id (`message-list.ts`). Different consumers need different views:

| View | Contents | Used for |
|---|---|---|
| `forLlm()` | system + working-memory block, then all messages (filtered) | the LLM call |
| `turnDelta()` | input ∪ response | what to persist to memory |
| `responseDelta()` | response only | what the caller sees (no echoed input) |

Two non-obvious details that prevent real bugs:
- **`stripOrphanedToolMessages`** runs before every LLM call — an incomplete tool-call/result pair (e.g. after a suspend) must never reach the model or the provider errors.
- **Monotonic `createdAt`** — live messages get `max(hint, lastCreatedAt + 1)` so batched tool results in the same millisecond keep a stable order; reload-by-timestamp then matches insertion order instead of shuffling.

Serialization stores the id arrays per set, so after suspend/resume the history-vs-turn classification is fully restored.

---

## Event bus + cancellation

A small **custom in-process pub/sub** — no `EventEmitter`, RxJS, or Node events. Typed, synchronous, ~160 lines:

```ts
class AgentEventBus {
  private handlers = new Map<AgentEvent, Set<AgentEventHandler>>();
  private controller = new AbortController();

  on(event: AgentEvent, h: AgentEventHandler) { /* add to the Set */ }
  off(event: AgentEvent, h: AgentEventHandler) { /* remove */ }
  emit(data: AgentEventData) { this.handlers.get(data.type)?.forEach((h) => h(data)); }  // sync, inline
}
```

- **Typed** — `AgentEvent` enum keys + a discriminated-union `AgentEventData` payload, so a handler narrows on `data.type`.
- **Synchronous** — `emit()` calls handlers inline; no queue, no async dispatch, no persistence.
- **Fresh bus per run** — `createRuntime()` news up a bus for each `generate()`/`stream()`/`resume()`; the agent stores subscriptions in its own `agentHandlers` map and copies them onto the new bus, so `on()`/`off()` and `abort()` always target the active run.

```ts
agent.on(AgentEvent.ToolExecutionStart, ({ toolName, args }) => log(toolName, args));
agent.on(AgentEvent.TurnEnd, ({ message, toolResults }) => { /* … */ });
```

Lifecycle events: `AgentStart`/`AgentEnd`, `TurnStart`/`TurnEnd`, `ToolExecutionStart`/`End`, `SubAgentStarted`/`Completed`, `Error`.

**Internal consumers** (not just user code):
- **Stream mode** subscribes to tool/sub-agent events and turns them into `StreamChunk`s on the HTTP stream — the bridge between the runtime and the wire.
- **Tools** receive an `emitEvent` hook in their context so platform tools can publish.
- **Sidecars don't subscribe** — they're scheduled imperatively (see [agentic-patterns.md](../cheatsheets/agentic-patterns.md) sidecars); they only `emit` an `Error` with a `source` on failure.

The bus also **owns cancellation** (the `AbortController`), so `agent.abort()` and the subscriptions target the same run:

- The signal goes straight into `generateText`/`streamText` as `abortSignal` → in-flight HTTP cancels promptly.
- `resetAbort(externalSignal?)` runs per-run: fresh controller, forwards a caller-provided `AbortSignal` into the internal one (external cancellation composes); `createAbortScope()` gives resume flows their own scope.
- The loop also checks `bus.isAborted` at batch boundaries.

> `agent.middleware()` stores handlers but isn't wired to the bus yet — a placeholder for future guardrails/HITL composition.

---

## Sub-agent delegation

A built-in `delegate_subagent` tool lets the model spawn child runs. The inline child runner reuses the parent's model and a *filtered* tool surface.

```ts
new Agent('researcher')
  .tool(searchTool)
  .tool(createDelegateSubAgentTool({ policy: { maxChildren: 2 } }));
```

- The model calls `delegate_subagent` with `subAgentId: "inline"`; the SDK runs a fresh child context using a shared delegated-task prompt.
- **`maxChildren`** is the parallel *batch width* for consecutive delegate calls — a parent can fan out N children at once even when ordinary tool concurrency is `1`.
- Children inherit the parent's tools **minus a blocklist** — no recursive `delegate_subagent`, no `write_todos`, no memory recall — so delegation can't loop or corrupt parent memory.
- A host can supply its own `runSubAgent` callback to route delegation elsewhere; both paths return the same output shape and emit the same lifecycle events.

---

## Cross-cutting product concerns

These all attach through the same builder/runtime seam rather than being bolted onto call sites:

- **Working memory** — when configured, the runtime injects an `update_working_memory` tool and renders the current state into the system prompt; the model reads it inline and writes via the tool (validated, async-persisted).
- **Episodic / observational memory** — background extract+reflect tasks summarise runs into a recallable store, exposed to the model as a `recall_memory` tool.
- **Guardrails** — input/output builders (`Guardrail('injection-detector').type('prompt-injection').strategy('block')`) plus PII redaction, applied around the loop.
- **Structured output** — `Output.object` + a Zod schema; the final result is parsed and surfaced on `finish`.
- **Evals** — LLM-judge scorers (`correctness`, `helpfulness`, `tool-call-accuracy`) run over agents + datasets via `evaluate()`.
- **Telemetry & cost** — the AI SDK's `experimental_telemetry` is wired through (OTel, optional LangSmith adapter), and token usage is priced from a provider catalog (`getModelCost`/`computeCost`).

---

## Patterns across frameworks

The same patterns recur across open-source ADKs — strong evidence they're inherent to the problem, not n8n's idiosyncrasies:

| Pattern | n8n `@n8n/agents` | Mastra | LangGraph / deepagents |
|---|---|---|---|
| Suspend / resume + HITL | branded suspend + `CheckpointStore` | workflow `suspend()` + storage | `interrupt()` + checkpointer |
| Memory layers | working + observation-log + episodic | working memory + **semantic recall** | `BaseStore` + checkpointer |
| Sub-agents | `delegate_subagent` (inline/maxChildren) | agents-as-tools / networks | `subagents` (deepagents), supervisor |
| Guardrails | input/output `Guardrail` builders | **processor pipeline** + tripwire | middleware (deepagents) |
| Tracing/cost | AI SDK telemetry + catalog pricing | OTel observability + scorers | LangSmith |
| Planning scratchpad | `write_todos` tool | — | todos middleware (deepagents) |

Views other ADKs add that are worth stealing:

- **Guardrails as a processor pipeline (Mastra).** Instead of standalone guardrail objects, Mastra runs `processInput` / `processOutputStream` / `processOutputResult` processors around the model, each able to `abort()` (a "tripwire"). Composable input/output middleware beats one-off checks — the same insight as deepagents composing agents from ordered **middleware** layers (deterministic ordering).
- **Workflows vs agents as distinct primitives (Mastra / LangGraph).** An *agent* is a model-driven loop; a *workflow* is a durable, explicit graph of steps (`createStep().then().branch().parallel()`, suspendable). Use a workflow when the control flow is known and you want determinism + durability; an agent when the model must decide. Many products need both.
- **Two-level memory scope: `resourceId` + `threadId` (Mastra).** A *thread* is one conversation; a *resource* is the user/tenant that owns many threads. Validating thread-owned-by-resource is the multi-tenancy guard. **Semantic recall** (embed past messages, retrieve relevant ones) sits alongside working memory. See [ai-persistence-patterns.md](ai-persistence-patterns.md).
- **Pluggable workspace backend (deepagents).** The agent's "filesystem" is an abstraction (`StateBackend` default; swap to real FS, an object store, or a sandbox like Daytona/Modal) — the same dependency-inversion idea as the `CheckpointStore` seam, applied to the agent's working files.
- **Middleware-composed agents (deepagents).** `createDeepAgent({ middleware, subagents, backend, tools })` builds the agent by layering middleware in a deterministic order — an alternative to n8n's subclass-`build()` injection for attaching cross-cutting behaviour.
- **A2A — agent-to-agent protocol (Mastra `a2a`, deepagents ACP).** Multi-agent over a wire protocol, so a sub-agent can be a *remote* service, not just an in-process call — the network-boundary version of sub-agent delegation.

> **Grounded:** Mastra `packages/core/src/{processors,workflows,memory}`, deepagents `libs/deepagents/src/{agent,backends,middleware}`, plus a production supervisor topology in [AWS's multi-agent Bedrock AgentCore guidance](https://github.com/aws-solutions-library-samples) (supervisor + specialist agents).

---

## Core design decisions

- **Build artifacts are plain data.** Builders emit `Built*` objects with no behaviour; the runtime is the only thing that *runs*. Testable, serializable, swappable.
- **Take over the agent loop.** Handing tools to the SDK without `execute` and looping yourself is what makes HITL, concurrency, and durable suspend possible. The SDK becomes a single-step `generate/stream` primitive.
- **One seam for infra.** Credentials, checkpoints, and provider choice all resolve in `build()` — extend it (subclass) rather than threading config through the API.
- **Suspension is a value, not an exception.** A branded return is easier to persist and resume than unwinding a stack; HITL reuses it verbatim.
- **Provider as a string + registry.** `"provider/model"` decouples user code from SDK provider packages and lets you lazy-load only what's used.
- **Provenance over counters.** Tracking message *sets* (not a `historyCount`) survives suspend/resume and yields clean LLM/persistence/response views.

---

## Tips

- Keep the underlying SDK out of the public surface — wrap `generateText`/`streamText` in one engine file; expose builders and `Built*` types only.
- Make `build()` the single resolution point and override it for platform concerns; never let user code call it.
- Model suspend/resume as branded data + a `CheckpointStore` interface; ship an in-memory default, swap in a durable store for production.
- Cap the loop (`MAX_LOOP_ITERATIONS`) and check abort at batch boundaries — runaway tool loops are the default failure mode.
- Validate provider credentials and tool inputs (Zod/JSON Schema) before the SDK sees them — fail with your errors, not the provider's.
- Thread one `AbortController`/event bus per run so `abort()`, cancellation, and subscriptions can't target a stale loop.
