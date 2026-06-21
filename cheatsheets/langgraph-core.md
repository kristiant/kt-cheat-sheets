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
