# AI Persistence Patterns

**What it is:** The canonical data *shapes* AI systems persist — how conversations, agent state, memory, vectors, and caches are modelled in DBs, vector stores, and key-value stores.

**Why people use it:** When you build an agent/RAG app you have to design its storage. These are the schemas production systems actually use, so you copy a proven shape instead of inventing one that breaks on resume, dedup, or multi-tenancy.

**Typically used for:** Designing the persistence layer for chatbots, agents, and RAG — message tables, checkpoint stores, memory stores, vector indexes.

> Grounded in [n8n](https://github.com/n8n-io/n8n)'s `@n8n/agents`, [LangGraph](https://github.com/langchain-ai/langgraphjs) (checkpoint/store), LangChain `BaseMessage`, and [turbopuffer](cheatsheets/turbopuffer.md). Mechanics live in [langgraph-core.md](../cheatsheets/langgraph-core.md), [langchain-core.md](../cheatsheets/langchain-core.md), [agentic-products.md](agentic-products.md); this doc is the *data shapes*.

---

## Messages & threads

A conversation is an append-only message log, scoped to a thread.

```ts
// messages
{
  id: string;            // STABLE id — enables dedup, replace-by-id, delete-by-id
  threadId: string;      // conversation scope
  role: "human" | "ai" | "system" | "tool";
  content: string | ContentBlock[];   // JSON; string or multimodal blocks
  toolCallId?: string;   // ToolMessage → the AIMessage tool_call it answers
  toolCalls?: Json;      // on an AIMessage
  metadata?: Json;
  createdAt: Date;       // monotonic + unique (see below)
  seq?: number;          // SQL tiebreaker
}

// threads
{ threadId: string; resourceId: string /* user/tenant */; title?: string; createdAt: Date }
```

Design choices that matter:

- **Stable `id` on every message** — so streaming can *replace* a partial, and graph state can *delete by id* (`RemoveMessage`). Not just an auto-increment.
- **Monotonic `createdAt`** — batched tool results in the same millisecond get identical timestamps; reload-by-time then shuffles. Assign `createdAt = max(hint, lastCreatedAt + 1)` so order is deterministic. Load with `ORDER BY createdAt, seq`.
- **`toolCallId` is the join** between an `AIMessage.tool_calls[i]` and its answering `ToolMessage`.

> Grounded: n8n `AgentMessageList` (the monotonic-`createdAt` fix), LangChain `BaseMessage`. See [langchain-core.md](../cheatsheets/langchain-core.md).

---

## Checkpoints & agent state

State snapshotted after each step, so a run can suspend, resume, and time-travel.

```ts
interface Checkpoint {
  v: number;
  id: string;                                 // uuid6 — time-sortable
  ts: string;
  channel_values: Record<string, unknown>;    // the full state
  channel_versions: Record<string, number>;
  versions_seen: Record<string, Record<string, number>>;
}
// stored as a tuple keyed by (thread_id, checkpoint_ns, checkpoint_id):
//   { config, checkpoint, metadata, parentConfig, pendingWrites }
```

For HITL / suspend-resume, the saved record also carries the *pending* work:

```ts
// n8n SerializableAgentState
{ status: "suspended" | …; messageList; pendingToolCalls; resumeData?; usage? }
```

Design choices:
- **Thread-scoped, append-only history** — `thread_id` keys the chain; `checkpoint_ns` scopes a subgraph. Keeping every checkpoint (not just latest) is what enables time-travel/fork.
- **`pendingWrites` / `pendingToolCalls`** — in-flight work captured at suspend so resume doesn't drop tool calls.
- **Store interface, not a table** — `suspend(runId, state)` / `resume(runId)` (load, don't delete) / `complete(runId)` (delete). Swap in-memory → Postgres without touching callers.

> Grounded: LangGraph `Checkpoint`/`BaseCheckpointSaver`, n8n `CheckpointStore`. See [langgraph-core.md](../cheatsheets/langgraph-core.md).

---

## Memory layers

Distinct layers by scope and lifetime — don't conflate them.

| Layer | Shape | Scope / lifetime |
|---|---|---|
| **Working memory** | a KV blob rendered into the system prompt | current run; small, always-in-context |
| **Observation log** | append-only entries (markdown bullets) + a cursor | per-thread; compressed conversation memory |
| **Episodic** | entries `{ content, embedding, scope, source, createdAt }` | long-term; recalled by similarity |
| **Cross-thread store** | `{ namespace: string[], key, value, createdAt, updatedAt }` | spans threads; user/app facts |

**Cursors** are how derived layers track what they've processed — a watermark on the *source*, not the derived store:

```ts
interface ObservationCursor {
  observationScopeId: string;     // usually threadId
  lastObservedMessageId: string;  // last message the observer summarised
  lastObservedAt: Date;
}
// episodic keeps its OWN cursor over observation rows: { lastIndexedObservationId, … }
```

`BaseStore` namespaces are hierarchical paths (`["memories", userId]`) — folder-like, and the unit of scoping/search.

> Grounded: n8n observation-log / episodic memory + cursors, LangGraph `BaseStore`, Mastra memory (working memory + **semantic recall**, scoped by `resourceId` → `threadId` with thread-owned-by-resource validation). See [agentic-products.md](agentic-products.md) (sidecar memory) and [langgraph-core.md](../cheatsheets/langgraph-core.md).

---

## Vectors & RAG storage

A vector row is an `id` + embedding + **the real content/metadata as attributes** (the vector is for search; attributes are the data).

```ts
// a row in a namespace/collection
{
  id: "policy.pdf#chunk-3",
  vector: number[],          // the embedding (search key)
  text: string,              // real content — for BM25 + display
  source: string,
  page: number,
  parentId?: string,         // parent-document: link a child chunk to its parent
}
```

Design choices:
- **Namespace / prefix per tenant** (`tenants/{id}/…`) — the isolation, lifecycle, and deletion boundary.
- **`distance_metric` is fixed per namespace** at creation (cosine vs euclidean).
- **Parent-document split** — small child chunks live in the vector store keyed by `parentId`; the larger parents live in a separate docstore (a `byteStore`), fetched after retrieval.
- Store `source`/`page`/`chunkId` so you can cite and filter.

> Grounded: [turbopuffer.md](../cheatsheets/turbopuffer.md), pgvector, LangChain `ParentDocumentRetriever`. Retrieval usage in [rag.md](../cheatsheets/rag.md).

---

## Caches

| Cache | Key → value | Where |
|---|---|---|
| **LLM response** | `hash(prompt + model + params)` → completion | exact-match skip of the API call |
| **Semantic** | embedding → answer (lookup by similarity ≥ threshold) | near-duplicate questions hit one entry |
| **Node cache** | `nodeName + hash(input)` → output (with TTL) | skip re-running a deterministic graph node |
| **Provider prompt cache** | stable prompt *prefix* → server-side KV | provider-side; order prompts prefix-stable to hit it |

Design choice: exact caches are cheap but brittle; **semantic caches** raise hit rate at the cost of a similarity threshold you must tune (too loose → wrong answers).

> Grounded: LangChain `InMemoryCache`, LangGraph `cachePolicy`. Prompt-cache ordering in [agentic-patterns.md](agentic-patterns.md).

---

## Graph structures

**Agent graph** (LangGraph `StateGraph`) — persisted as two separate things:
- the **graph definition** (nodes → runnables, edges, channels) — code, not data
- the **per-run state** (channels) — the checkpoints above

**Knowledge graph** (GraphRAG):

```ts
{ entities:  { id, type, name, embedding } }
{ relations: { source, target, type, weight } }
{ communities: { id, members: entityId[], summary } }   // clustered + summarised for retrieval
```

Traversed with BFS/DFS at query time (see [ai-algorithms.md](../cheatsheets/ai-algorithms.md)); entities also embedded so you can vector-search *into* the graph.

---

## Cross-cutting design rules

- **Stable ids on everything** — dedup, replace, delete-by-id, and joins (`toolCallId`, `parentId`) all depend on them.
- **A scope key is your isolation + deletion boundary** — `threadId`, `resourceId`, vector namespace. Decide it before you write a row; migrations are painful.
- **Monotonic, unique timestamps** — or reload/pagination order is non-deterministic.
- **Separate hot / durable / derived** — per-thread checkpoints (hot, resumable), cross-thread store (durable facts), observation/episodic (derived, regenerable). Different lifetimes, different stores.
- **Persist behind an interface** (`CheckpointStore`, `BaseStore`) — start in-memory, swap to Postgres/Redis without touching call sites.
- **Keep the local impl at parity with the cloud one.** If you ship an in-memory dev store and a real one (DynamoDB/Postgres), they must agree on validation order *and error types* — or code written against the local store breaks in prod. Enforce with parity tests. (AWS Blocks does exactly this via conditional-export `mock`/`aws` entries + `parity.test.ts` — see [nodejs.md](../cheatsheets/nodejs.md) / [jest-vitest.md](../cheatsheets/jest-vitest.md).)
- **Conditional writes for concurrency.** Use `ifNotExists` / compare-and-set (DynamoDB condition expressions, a `version` column) rather than read-then-write — it's how `LastValue`-style "one writer wins" and idempotent upserts stay correct under concurrent steps. Grounded: AWS Blocks `KVStore` `ConditionalWriteOptions`.
