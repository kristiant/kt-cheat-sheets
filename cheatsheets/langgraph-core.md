# LangGraph Core — Internals

**What it is:** The internals of `@langchain/langgraph` — the graph primitives (state, nodes, edges, channels) and the Pregel execution engine that drive stateful, cyclic agent workflows.

**Why people use it:** Understanding the primitives makes graphs debuggable — you're really building a state machine whose nodes return partial updates merged through channel reducers each superstep.

**Typically used for:** Custom agent control flow, multi-actor graphs, and anything needing loops, branching, persistent state, or human-in-the-loop.

> Usage-level patterns are in [langgraph.md](langgraph.md) and [agentic-patterns.md](agentic-patterns.md); this sheet is the internals. Source: `@langchain/langgraph`.

---

## Graph primitives

### `START` / `END` (`constants.ts`)

```ts
const START = "__start__";   // where input enters
const END = "__end__";       // where execution terminates
```

Virtual sentinel nodes — not real nodes you implement. Every graph needs at least one edge from `START` and one to `END`.

### `StateGraph` — the builder

```ts
const graph = new StateGraph(MyState)
  .addNode("agent", agentFn)
  .addNode("tools", toolsFn)
  .addEdge(START, "agent")
  .addEdge("tools", "agent")
  .addConditionalEdges("agent", router)
  .compile();
```

Extends the lower-level `Graph`; the difference is it carries **shared state (channels)** flowing through all nodes. Accepts state as `Annotation.Root(...)` (standard), a Zod schema (modern), or `{ state, input, output }` for separate input/output shapes.

### `Annotation` — defining state

```ts
const MyState = Annotation.Root({
  messages: Annotation<BaseMessage[]>({
    reducer: (existing, update) => [...existing, ...update],
    default: () => [],
  }),
  topic: Annotation<string>,   // no reducer → last-write-wins
});
```

Each field is a **channel**: `Annotation<T>` with no args → a `LastValue` channel (overwrite); with a `reducer` → a `BinaryOperatorAggregate` channel (accumulate). Two type slots come off the root:

- `typeof MyState.State` — full state shape (type node inputs)
- `typeof MyState.Update` — partial update shape (type node outputs)

### Nodes

A node is any `RunnableLike` — function, Runnable, or async fn. It receives the **full state** and returns a **partial update** (only the keys it returns are merged):

```ts
const agentNode = async (state: typeof MyState.State) => {
  return { messages: [new AIMessage("...")] };   // partial — merged via the channel reducer
};
graph.addNode("agent", agentNode);
```

`AddNodeOptions`: `metadata` (tracing), `ends` (pre-declare reachable nodes for visualisation/validation).

### Edges

Fixed, unconditional routing:

```ts
graph.addEdge("tools", "agent");   // after tools, always go to agent
graph.addEdge("agent", END);
```

Plain `Graph` allows one outgoing edge per node; `StateGraph` lifts this (fan-in via waiting edges).

### Conditional edges

Dynamic routing — a function of state picks the next node:

```ts
graph.addConditionalEdges("agent", (state) =>
  state.messages.at(-1)?.tool_calls?.length ? "tools" : END,
);
```

The router returns a node name, `END`, a `Send` (map-reduce — see control flow), or an **array** of these (fan-out). `BranchPathReturnValue = string | Send | (string | Send)[]`.

### `MessagesAnnotation` — prebuilt chat state

```ts
const graph = new StateGraph(MessagesAnnotation);
// ≡ Annotation.Root({ messages: Annotation<BaseMessage[]>({ reducer: messagesStateReducer, default: () => [] }) })
```

`messagesStateReducer` isn't naive concat — it handles `RemoveMessage` (delete by id) and dedupes by message id. Use it for any chat-style agent.

### `.compile()` → `CompiledGraph`

```ts
const app = graph.compile({
  checkpointer,                  // persistence (next topic)
  interruptBefore: ["tools"],    // pause before a node runs
  interruptAfter: ["agent"],
});
```

Returns a `CompiledGraph` extending **`Pregel`** — itself a `Runnable`, so `.invoke()`/`.stream()`/`.batch()` all work. Pregel is the execution engine: **superstep-based** (à la Google Pregel) — nodes run in parallel within a superstep, then state merges via channel reducers before the next.

### Mental model

```
StateGraph (builder)
  ├─ nodes:    name → RunnableLike(state) → partial update
  ├─ edges:    fixed routes
  ├─ branches: conditional routes
  └─ .compile() → CompiledGraph (Pregel)
        ├─ .invoke(input)      → final state
        ├─ .stream(input)      → stream of state updates
        └─ .getState(config)   → current checkpoint state
```

Each superstep: **identify** nodes to run (from last step's edges) → **run** them (parallel) → **merge** returns into state via channel reducers → **evaluate** outgoing edges → repeat until `END`.

---

## State management

### Channels — the underlying primitive

State isn't a plain object — **every key is a channel**. A channel controls how it accepts updates (`update(values[])`), what it returns (`get()`), and how it serialises (`checkpoint()` / `fromCheckpoint()`).

| Channel | Behaviour | Created by |
|---|---|---|
| `LastValue` | most recent write; **errors if >1 writer in a step** | `Annotation<T>` (no reducer) |
| `BinaryOperatorAggregate` | folds updates with a reducer | `Annotation<T>({ reducer })` |
| `EphemeralValue` | holds a value for one step, then auto-clears | internal / `Topic` |
| `Topic` | pub/sub — collects all values published in a step into an array, optional dedupe | internal / advanced |

(Annotation basics — `.State`/`.Update` slots, last-write-wins vs reducer — are in [Graph primitives](#graph-primitives) above.) A third slot, `typeof MyState.Node`, types node functions.

### StateSchema — the Zod alternative

```ts
import { StateSchema, ReducedValue, UntrackedValue } from "@langchain/langgraph";

const MyState = new StateSchema({
  count: z.number().default(0),                 // → LastValue
  messages: ReducedValue(z.array(...), reducer), // → BinaryOperatorAggregate
  scratch: UntrackedValue(z.string()),          // not checkpointed
});
```

Same `.State` / `.Update` / `.Node` slots as `AnnotationRoot`; converts to channels via `getChannels()`. Adds Zod validation on state values.

### `messagesStateReducer` in detail

Why `MessagesAnnotation` is more than array-append — it:

1. Normalises both sides to `BaseMessage[]`, assigning missing ids.
2. **Replace by id** — a right-side message with an existing id replaces it (streaming updates).
3. **Delete by id** — `RemoveMessage({ id })` removes that message.
4. **Delete all** — `RemoveMessage({ id: REMOVE_ALL_MESSAGES })` wipes everything before it.
5. **Append** — genuinely new ids are appended.

### Input / output schemas — filtering state

```ts
new StateGraph({ state: FullAnnotation, input: InputAnnotation, output: OutputAnnotation });
```

Full state (all channels) runs internally; `input` is merged at `START`, `output` projected at `END` — exposing only a subset externally.

### Private / untracked state

`UntrackedValue` (StateSchema) → an `UntrackedValueChannel`: present during execution but **not persisted to checkpoints** — for ephemeral working memory, caches, or data too large to serialise. Annotation-based state has no direct equivalent; stash large data in a `ToolMessage.artifact` instead.

### How state flows through Pregel

```
each superstep:
  1. nodes run → return partial updates  { messages: [...], topic: "foo" }
  2. Pregel groups all updates per channel across the nodes that ran
  3. channel.update(updatesForThisKey[]):
       LastValue                → throws if >1 writer; stores the value
       BinaryOperatorAggregate  → folds all updates via the reducer
       EphemeralValue           → stores, clears next step
  4. new state = channel.get() per key
  5. checkpoint saved (if a checkpointer is configured)
  6. conditional edges evaluated on the NEW state → next nodes
```

### Mental model

```
Node A returns { messages: [msg1] }
Node B returns { messages: [msg2] }     ← same step, ran in parallel
  messagesStateReducer([msg1], [msg2]) → [msg1, msg2]   (reducer accumulates)
  a LastValue channel would THROW here  (two writers in one step)
```

State is never passed by reference — each step produces a **new immutable snapshot**. Checkpointing saves these snapshots, enabling time-travel and resumption.

---

## Control flow

Conditional edges (in [Graph primitives](#graph-primitives)) are the basic router; returning an **array** fans out (multiple nodes run next step). The rest of the toolkit:

### `Command` — route + update in one return

A node can return a `Command` that both updates state and routes — no separate conditional edge:

```ts
const nodeA = async (state) =>
  new Command({
    update: { foo: "updated" },                       // state update
    goto: Math.random() > 0.5 ? "nodeB" : "nodeC",    // routing (name | Send | array)
  });
graph.addNode("nodeA", nodeA, { ends: ["nodeB", "nodeC"] });
```

Fields: `update`, `goto`, `resume` (for `interrupt()`), `graph` (use `Command.PARENT` to route in the parent from a subgraph). Tools can return a `Command` directly (`lc_direct_tool_output`), so **a tool can drive agent routing**.

### `interrupt()` — human-in-the-loop pause

```ts
const reviewNode = async (state) => {
  const decision = interrupt({ question: "Approve?", data: state.plan });
  // PAUSES here — graph suspends, checkpoint saved
  return decision === "approve" ? { approved: true } : { approved: false };
};
```

How it works: `interrupt(value)` checks the scratchpad for a stored resume value; if none, it **throws `GraphInterrupt`** (caught by Pregel, not you) → graph suspends with a pending interrupt. Re-invoke with `Command({ resume: "approve" })` → graph **replays to the interrupt point**, `interrupt()` now returns the resume value, execution continues.

- Requires a checkpointer. **Don't wrap `interrupt()` in try/catch** — `GraphInterrupt` must propagate.
- Multiple `interrupt()` calls in one node work — handled sequentially across invocations.

### `interruptBefore` / `interruptAfter` — static interrupts

Declared at compile time, no code changes — pause to inspect:

```ts
graph.compile({ interruptBefore: ["tools"], interruptAfter: ["agent"] });
```

Resume by re-invoking (or `Command({ resume: undefined })`) — no resume value needed; these just pause, they're not awaiting input.

### `Send` — dynamic fan-out / map-reduce

```ts
const continueToJokes = (state) =>
  state.subjects.map((subject) => new Send("generate_joke", { subject }));

graph.addConditionalEdges(START, continueToJokes);
graph.addEdge("generate_joke", END);
```

`Send(node, args)` spawns a separate, parallel invocation of `node` with a **custom input** (not the full state). Results merge back via the collecting channel's reducer. Unlike a string route, `Send` passes arbitrary per-invocation input — the map-reduce primitive.

### `Overwrite` — bypass a reducer

Force-replace a reducer-channel value instead of accumulating:

```ts
return { messages: new Overwrite(["replacement"]) };   // ignores the reducer
```

One `Overwrite` per channel per superstep — more throws `InvalidUpdateError`.

### Node policies — retry / timeout / error handling

```ts
graph.addNode("agent", agentFn, {
  retryPolicy: { maxAttempts: 3, initialInterval: 500, backoffFactor: 2,
    retryOn: (e) => e instanceof RateLimitError },
  timeout: 30_000,
  errorHandler: (state, { error }) => ({ lastError: error.message }),  // record + continue
});
```

`errorHandler` runs *instead of* propagating — record the failure in state and carry on. Graph-wide defaults via `setNodeDefaults()`; per-node overrides win.

### Subgraphs

A compiled graph is a Runnable, so it's a valid node:

```ts
const subgraph = new StateGraph(SubState).addNode(/* … */).compile();
graph.addNode("sub", subgraph);
```

State keys must **overlap** for values to pass through. Route from a subgraph back to the parent with `Command({ goto: "parentNode", graph: Command.PARENT })`.

### Mental model — pick the mechanism

| Need | Use |
|---|---|
| Fixed next step | `addEdge(a, b)` |
| State-based routing | `addConditionalEdges(a, routerFn)` |
| Route **and** update together | return `new Command({ update, goto })` |
| Pause for human input | `interrupt(q)` → resume via `Command({ resume })` |
| Parallel invocations / map-reduce | return `[new Send("node", args1), ...]` |
| Force-replace a reducer value | return `{ key: new Overwrite(value) }` |

---

## Persistence & memory

### Checkpoint — what gets saved

A snapshot of all channel values after each superstep:

```ts
interface Checkpoint {
  v: number;                                  // format version
  id: string;                                 // uuid6 — sortable by time
  ts: string;                                 // ISO timestamp
  channel_values: Record<string, unknown>;    // full state
  channel_versions: Record<string, number>;   // version per channel
  versions_seen: Record<string, Record<string, number>>;  // per node → channel → version
}
```

A `CheckpointTuple` wraps it with config, metadata, parent config, and **pending writes** (in-flight when the graph suspended).

### `BaseCheckpointSaver`

Four methods: `getTuple(config)`, `list(config, opts?)`, `put(config, checkpoint, metadata, versions)`, `putWrites(config, writes, taskId)`, `deleteThread(threadId)`.

| Saver | Package | Use |
|---|---|---|
| `MemorySaver` | `@langchain/langgraph-checkpoint` | dev/testing only |
| `SqliteSaver` | `…-checkpoint-sqlite` | local persistence |
| `PostgresSaver` | `…-checkpoint-postgres` | production SQL |
| `MongoDBSaver` | `…-checkpoint-mongodb` | production document |
| `RedisSaver` | `…-checkpoint-redis` | production cache-backed |

### Thread scoping — `thread_id`

The checkpointer key. Same `thread_id` → shared history; different → isolated.

```ts
await app.invoke(input, { configurable: { thread_id: "user-123" } });  // continues that thread
```

`checkpoint_ns` further scopes to a subgraph within a thread.

### Time-travel — `getState` / `getStateHistory` / `updateState`

```ts
const s = await app.getState({ configurable: { thread_id: "123" } });
// s.values (channels) · s.next (nodes that would run) · s.tasks · s.metadata

for await (const snap of app.getStateHistory({ configurable: { thread_id: "123" } })) { /* newest first */ }

// fork: re-run from a past checkpoint
await app.invoke(input, { configurable: { thread_id: "123", checkpoint_id: past.id } });

// inject state manually (asNode = whose turn it looks like)
await app.updateState({ configurable: { thread_id: "123" } }, { messages: [new HumanMessage("fix")] }, "agent");
```

### `BaseStore` — cross-thread / long-term memory

Checkpoints are **per-thread**; `BaseStore` is shared across threads. Namespaces are hierarchical paths (folder-like):

```ts
await store.put(["memories", userId], "pref-1", { color: "blue" });
const item = await store.get(["memories", userId], "pref-1");          // → { value, key, namespace, createdAt, updatedAt }
await store.search(["memories", userId], { filter: { color: "blue" }, limit: 5 });
await store.search(["documents"], { query: "user preferences", limit: 10 });  // semantic, if vectors supported
```

Wired at compile time (`graph.compile({ checkpointer, store })`); reached via `runtime.store` in tools (ToolRuntime) or `config.store` in nodes.

### Node cache — `cachePolicy`

Per-node result caching — same input + same node → cached output, skip the run:

```ts
graph.addNode("expensive", expensiveFn, { cachePolicy: { ttl: 60_000 } });
graph.compile({ cache: new InMemoryCache() });   // backing store
```

Keyed by node input + node name; for deterministic nodes with costly LLM calls.

---

## Prebuilt patterns

### `ToolNode`

A ready-made node that runs all tool calls from the last `AIMessage`:

```ts
const toolNode = new ToolNode([tool1, tool2]);
graph.addNode("tools", toolNode);
```

Per invocation it: finds the last `AIMessage` → runs its `tool_calls` **in parallel** → returns one `ToolMessage` per call. With `handleToolErrors: true` (default) it catches errors as `ToolMessage({ status: "error" })`, **re-throws `GraphInterrupt`** (so `interrupt()` in tools still works), and injects full `ToolRuntime` (state/store/writer/config) into each tool.

### `toolsCondition`

The router for the edge out of the agent node — `"tools"` if the last message has `tool_calls`, else `END`. Works with both `BaseMessage[]` and `{ messages }` state shapes.

```ts
graph.addConditionalEdges("agent", toolsCondition);
```

### `createReactAgent`

Builds a complete ReAct agent graph in one call:

```ts
const agent = createReactAgent({
  llm: model,
  tools: [tool1, tool2],
  prompt: "You are a helpful assistant",     // string | SystemMessage | fn | Runnable
  checkpointer,
  store,
  responseFormat: z.object({ answer: z.string() }),  // structured final output (adds a step)
  stateSchema,                                // custom state (extends MessagesAnnotation)
});
```

It assembles `START → agent → toolsCondition → tools → agent → … → END`, where **agent** = prompt → `model.bindTools(tools)` and **tools** = `ToolNode(tools)`. Notable params: `prompt`/`stateModifier` (pre-model state transform), `preModelHook`/`postModelHook`, `responseFormat`, `version: "v1" | "v2"` (v2 runs tools as a subgraph).

### Mental model — how it all connects

```
createReactAgent({ llm, tools, checkpointer, store })
  → CompiledStateGraph (MessagesAnnotation)
     ├─ thread_id → checkpointer → per-conversation history
     ├─ store     → cross-thread memory
     ├─ "agent"   → prompt + model.bindTools() → AIMessage (tool_calls or final answer)
     ├─ toolsCondition → "tools" | END
     └─ "tools"   → ToolNode: runs tool_calls in parallel, each gets state+store via ToolRuntime → ToolMessage[]
```
