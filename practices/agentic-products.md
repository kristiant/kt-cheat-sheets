# Building an Agentic Product on an Agent SDK

**What it is:** The architecture for turning a low-level agent/LLM SDK (Vercel AI SDK) into a *product-grade* agent runtime — a declarative authoring layer, an execution engine that owns the loop, and the wiring for suspend/resume, sub-agents, memory, and observability.

**Why people use it:** The AI SDK gives you `generateText`/`streamText` and a basic tool loop. A product needs more: human-in-the-loop, durable checkpoints, concurrency control, multi-provider models, guardrails, evals, and telemetry. This is how you layer those on without leaking the SDK everywhere.

**Typically used for:** Agent platforms, internal agent runtimes, anything that runs *other people's* agents and must be controllable, resumable, and observable.

> Grounded in [n8n](https://github.com/n8n-io/n8n)'s `@n8n/agents` package (`packages/@n8n/agents`), an agent SDK built directly on the [Vercel AI SDK](https://sdk.vercel.ai). File paths below are relative to that package.

---

## The two-layer split: SDK vs runtime

The single most important decision: separate the **declarative authoring layer** (`sdk/`) from the **execution engine** (`runtime/`). The AI SDK is touched *only* in the runtime.

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

The AI SDK can run a tool loop itself (via `maxSteps`/auto-executed tools). This runtime **deliberately takes that over.** Tools are handed to the SDK *without* an `execute` function, so the model proposes calls but the runtime executes them.

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

Owning the loop is what unlocks everything else: per-batch concurrency, suspend mid-batch, HITL gates, abort checks between batches, custom persistence. You can't get those if the SDK drives the loop.

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

Patterns worth copying:
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

One `AgentEventBus` per agent, shared between the builder (`on()`) and the runtime (`emit()`), created when the SDK wires the runtime.

```ts
agent.on(AgentEvent.ToolExecutionStart, ({ toolName, args }) => log(toolName, args));
agent.on(AgentEvent.TurnEnd, ({ message, toolResults }) => { /* … */ });
```

Lifecycle events: `AgentStart`/`AgentEnd`, `TurnStart`/`TurnEnd`, `ToolExecutionStart`/`ToolExecutionEnd`, `Error`. The bus also **holds the `AbortController`**, so `agent.abort()` and the event subscriptions always target the same run:

- The signal is passed straight into `generateText`/`streamText` as `abortSignal` → in-flight HTTP cancels promptly.
- `resetAbort(externalSignal?)` runs per-run: fresh controller, and forwards a caller-provided `AbortSignal` into the internal one (so external cancellation composes).
- The loop also checks `bus.isAborted` at batch boundaries.

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

## Design decisions worth stealing

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
