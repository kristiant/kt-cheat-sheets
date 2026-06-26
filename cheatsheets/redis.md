# Redis (for AI engineering)

**What it is:** An in-memory key-value store with rich data structures (hashes, lists, sorted sets, streams) and, via Redis Stack, **vector search** and JSON. Sub-millisecond, optionally persistent.

**Why people use it:** It's the fastest place to cache, rate-limit, queue, and coordinate. In AI apps it's the semantic cache, the vector store, the chat-history store, the rate limiter, and the agent event bus — often all at once.

**Typically used for:** LLM response caching, vector/semantic search for RAG, per-tenant rate limiting, conversation memory, distributed locks, and streaming/job queues.

> TS-first (node-redis v5). Pure-vector-DB alternative: [turbopuffer.md](turbopuffer.md). Where these patterns sit in a pipeline: [rag.md](rag.md), [agentic-patterns.md](../practices/agentic-patterns.md).

---

## Mental model

A flat keyspace where each key holds a typed value; everything lives in RAM (fast, bounded by memory), **single-threaded** (one slow command blocks all), with optional disk persistence. Most AI uses lean on **TTLs** (auto-expiry) and the right data structure per job.

```bash
npm i redis            # node-redis v5 (official). Or `ioredis` for cluster-heavy setups.
```

```ts
import { createClient } from "redis";

const client = createClient({ url: process.env.REDIS_URL });
await client.connect();

await client.set("k", "v", { EX: 60 });   // value + 60s TTL
const v = await client.get("k");
```

---

## Fundamentals (node-redis)

**The client** is your connection to the server (from `createClient().connect()`). Every command is an `async` method on it — they all return promises, so `await` everything.

**Core commands are flat on `client`; module commands are namespaced.** node-redis camelCases the raw Redis command name:

```ts
await client.set(...)        // SET
await client.hSet(...)       // HSET   (HSET → hSet)
await client.zAdd(...)       // ZADD
await client.ft.search(...)  // FT.SEARCH  → search module, under client.ft
await client.json.set(...)   // JSON.SET   → JSON module, under client.json
```

| Namespace | Module | Commands |
|---|---|---|
| *(flat on `client`)* | core Redis | `set`, `get`, `hSet`, `incr`, `rPush`, `zAdd`, `xAdd`, … |
| `client.ft` | RediSearch (Query Engine) | `ft.create`, `ft.search` — full-text **and vector/KNN** |
| `client.json` | RedisJSON | `json.set`, `json.get` |
| `client.ts` | RedisTimeSeries | `ts.add`, `ts.range` |
| `client.bf` / `client.cf` | Bloom / Cuckoo filter | `bf.add`, `bf.exists` |

> ⚠️ **The module namespaces (`ft`/`json`/…) require Redis Stack** (or Redis 8+, which bundles the modules) — they error on vanilla Redis. Run `docker run -p 6379:6379 redis/redis-stack` for local dev with vector search.

**Connection lifecycle** — make one client, reuse it everywhere (module scope), handle errors, and `duplicate()` only for a blocking subscriber:

```ts
const client = createClient({ url: process.env.REDIS_URL });
client.on("error", (err) => logger.error("redis", err));   // attach BEFORE connect
await client.connect();
// … reuse `client` for the app's lifetime …
await client.quit();   // graceful shutdown (flushes, closes)
```

**Key naming** — there are no tables; **convention is `:`-delimited namespaces** so keys are scannable and groupable. Put the scope first (the same isolation/deletion boundary idea as everywhere else):

```
chat:{threadId}        rl:{tenantId}:{minute}        doc:{id}        lock:{runId}
```

**Values are binary-safe** — strings, JSON (`JSON.stringify`), or raw bytes (vectors as a `Float32Array` buffer). **`EX`/`PX`/`EXPIRE`** set a TTL; `TTL key` checks remaining time; `PERSIST` removes it.

---

## Core structures (quick ref)

| Structure | Commands | AI use |
|---|---|---|
| String | `SET`/`GET`/`INCR` | response cache, counters, rate limits |
| Hash | `HSET`/`HGETALL` | a record (config, a cached row) |
| List | `RPUSH`/`LRANGE` | chat history (append + window) |
| Sorted set | `ZADD`/`ZRANGE` | ranked results, leaderboards, time series |
| Stream | `XADD`/`XREAD` | event log, durable job queue |
| Pub/Sub | `PUBLISH`/`SUBSCRIBE` | token streaming, fan-out (ephemeral) |
| Vector (Stack) | `FT.CREATE`/`FT.SEARCH` | semantic search, RAG, semantic cache |

---

## AI engineering uses

### LLM response cache (exact match)

Key on a hash of the prompt + model + params; skip the API call on a hit.

```ts
import { createHash } from "node:crypto";

const key = "llm:" + createHash("sha256").update(model + "|" + prompt).digest("hex");
const hit = await client.get(key);
if (hit) return JSON.parse(hit);

const out = await callModel(prompt);
await client.set(key, JSON.stringify(out), { EX: 3600 });   // 1h TTL
```

### Vector store / semantic search (Redis Stack)

Create an index over a vector field, then KNN search. Powers RAG retrieval *and* semantic caching.

```ts
import { SchemaFieldTypes, VectorAlgorithms } from "redis";

await client.ft.create("idx:docs", {
  embedding: {
    type: SchemaFieldTypes.VECTOR,
    ALGORITHM: VectorAlgorithms.HNSW,   // or FLAT for small/exact
    TYPE: "FLOAT32", DIM: 1536, DISTANCE_METRIC: "COSINE",
  },
  text: { type: SchemaFieldTypes.TEXT },
}, { ON: "HASH", PREFIX: "doc:" });

// store: vector as a raw Float32 buffer
await client.hSet("doc:1", {
  embedding: Buffer.from(new Float32Array(embedding).buffer),
  text: "Refunds are issued within 5 business days.",
});

// KNN search
const res = await client.ft.search("idx:docs", "*=>[KNN 4 @embedding $vec AS score]", {
  PARAMS: { vec: Buffer.from(new Float32Array(queryEmbedding).buffer) },
  SORTBY: "score", DIALECT: 2, RETURN: ["text", "score"],
});
```

> Higher-level: **RedisVL** (`redis-vl`, an AI-native TS client) and LangChain's `RedisVectorStore` (`@langchain/redis`) wrap this. Hybrid search: combine the KNN with a filter expression (`@year:[2024 2024]=>[KNN ...]`).

### Semantic cache

The same vector index, used as a cache: embed the query, KNN against cached Q→A pairs, return the answer if the top score clears a threshold.

```ts
const [best] = (await client.ft.search("idx:cache", "*=>[KNN 1 @embedding $vec AS score]",
  { PARAMS: { vec: buf(qEmbedding) }, SORTBY: "score", DIALECT: 2, RETURN: ["answer", "score"] })).documents;
if (best && Number(best.value.score) <= 0.1) return best.value.answer;   // cosine distance ≤ 0.1
```

### Rate limiting (per-tenant LLM quotas)

`INCR` a per-window counter, set TTL on first hit (fixed window). The #1 guard against `429`s and runaway spend.

```ts
const k = `rl:${tenantId}:${Math.floor(Date.now() / 60_000)}`;   // per-minute bucket
const n = await client.incr(k);
if (n === 1) await client.expire(k, 60);
if (n > 100) throw new Error("rate limit exceeded");
```

### Conversation memory (chat history)

A list per thread; window the last N for the model (the sliding-window strategy — see [agentic-patterns.md](../practices/agentic-patterns.md)).

```ts
await client.rPush(`chat:${threadId}`, JSON.stringify(message));
await client.expire(`chat:${threadId}`, 86_400);
const recent = await client.lRange(`chat:${threadId}`, -20, -1);   // last 20 messages
```

> LangChain's `RedisChatMessageHistory` and LangGraph's `RedisSaver` (`@langchain/langgraph-checkpoint-redis`) persist agent threads/checkpoints on Redis. See [ai-persistence-patterns.md](../practices/ai-persistence-patterns.md).

### Distributed lock (coordinate agent/sidecar runs)

`SET NX PX` claims a lock with a TTL; release with a token check so you don't delete someone else's lock.

```ts
const token = crypto.randomUUID();
const got = await client.set(`lock:${runId}`, token, { NX: true, PX: 30_000 });
if (!got) return;                        // someone else holds it (skip / retry)
try { /* exclusive work */ }
finally {
  // atomic check-and-release (Redlock pattern)
  await client.eval(`if redis.call('get',KEYS[1])==ARGV[1] then return redis.call('del',KEYS[1]) end`,
    { keys: [`lock:${runId}`], arguments: [token] });
}
```

### Streams & pub/sub (events, queues, token streaming)

```ts
// Stream — durable, replayable agent event log / job queue
await client.xAdd("agent:events", "*", { type: "tool_call", tool: "search" });
const evs = await client.xRead({ key: "agent:events", id: "$" }, { BLOCK: 5000 });

// Pub/Sub — ephemeral fan-out (stream LLM tokens to subscribers)
const sub = client.duplicate(); await sub.connect();
await sub.subscribe(`stream:${runId}`, (chunk) => process.stdout.write(chunk));
await client.publish(`stream:${runId}`, token);
```

> For real job queues use **BullMQ** (built on Redis streams) — retries, concurrency, scheduling out of the box.

---

## Patterns from production repos

Real Redis usage from locally-studied codebases — the key schemas and gotchas that don't show up in tutorials.

### Single-leader election (n8n scaling)

Multi-instance deployments elect one leader to run singletons (cron, queue housekeeping). Claim with `NX` + a TTL; renew only if you still hold it (Lua), so a dead leader's key expires and another instance takes over.

```ts
// claim — only succeeds if no leader exists; TTL means a crashed leader auto-releases
const ok = (await client.set("n8n:leader", hostId, "EX", 15, "NX")) === "OK";
// renew (cron) — extend TTL ONLY if this host is still the leader (atomic, Lua)
await client.eval(
  `if redis.call('get',KEYS[1])==ARGV[1] then return redis.call('expire',KEYS[1],ARGV[2]) end`,
  { keys: ["n8n:leader"], arguments: [hostId, "15"] },
);
```

> Grounded: n8n's `leader-election-client.ts` (`SET … EX … NX` to claim, Lua to renew). Same shape as a distributed lock — a lock is just leadership over one resource (n8n's `collaboration:write-lock:${workflowId}`).

### Pub/Sub as a cross-instance bus (n8n scaling)

Beyond token streaming, pub/sub is how horizontally-scaled instances coordinate: a main publishes commands, workers subscribe and respond on a reply channel. Stateless fan-out — no instance needs to know who's listening.

```ts
// publisher (main): broadcast a command to all workers
await client.publish("n8n.commands", JSON.stringify({ type: "reload-workflow", id }));
// subscriber (worker): a DEDICATED connection (a subscribed conn can't run other commands)
const sub = client.duplicate(); await sub.connect();
await sub.subscribe("n8n.commands", (msg) => handle(JSON.parse(msg)));
```

> Grounded: n8n's `publisher.service.ts` / `subscriber.service.ts` — command, worker-response, and relay channels for multi-main/worker mode.

### Thread-scoped state keys (LangGraph Redis checkpointer)

Agent checkpoints key on the **conversation scope first**, with a sorted set tracking pending writes in order:

```
checkpoint:{threadId}:{checkpointNs}            → the state snapshot (hash)
write_keys:{threadId}:{checkpointNs}:{ckptId}   → a ZSET of pending write keys, ordered
```

> Grounded: `@langchain/langgraph-checkpoint-redis`. ZSETs aren't just leaderboards — here they keep tool-writes ordered within a checkpoint. See [ai-persistence-patterns.md](../practices/ai-persistence-patterns.md).

### ⚠️ Glob injection across tenants

If a **caller-controlled value** (a `threadId`, tenant id, session id from request input) goes straight into a key *or* a `KEYS`/`SCAN MATCH` pattern, a value of `"*"` becomes a wildcard. `KEYS "checkpoint:*:*"` then enumerates **every tenant**, and a delete-by-pattern wipes the database (CWE-943).

```ts
// validate caller-shaped key parts before embedding them
const GLOB = /[*?[\]\\]/;
function assertSafeKeyPart(field: string, v: string) {
  if (GLOB.test(v)) throw new Error(`unsafe Redis key part in "${field}": ${v}`);
}
assertSafeKeyPart("threadId", threadId);
await client.hGetAll(`checkpoint:${threadId}:${ns}`);
```

> Grounded: the LangGraph Redis saver rejects `* ? [ ] \` in any caller-influenced key field for exactly this reason (`:` is allowed — it's a literal delimiter, not a glob char).

> **Client note:** n8n uses **ioredis** (variadic args: `set(k, v, "EX", ttl, "NX")`); this sheet otherwise uses **node-redis** (options object: `set(k, v, { EX: ttl, NX: true })`). Same commands, different argument style.

---

## Practical recipes

**Cache-aside an embedding (don't re-embed the same text):**
```ts
const k = "emb:" + sha256(text);
const cached = await client.get(k);
const vec = cached ? JSON.parse(cached) : await embed(text);
if (!cached) await client.set(k, JSON.stringify(vec), { EX: 604_800 });
```

**Pipeline many writes (one round-trip):**
```ts
const p = client.multi();
for (const d of docs) p.hSet(`doc:${d.id}`, { embedding: buf(d.vec), text: d.text });
await p.exec();
```

**Set an eviction policy for a cache workload (redis.conf / CONFIG):**
```bash
redis-cli CONFIG SET maxmemory 2gb
redis-cli CONFIG SET maxmemory-policy allkeys-lru   # evict least-recently-used when full
```

**Inspect keys without blocking (never `KEYS *` in prod):**
```bash
redis-cli --scan --pattern 'chat:*' | head
```

**Drop a tenant's data:**
```bash
redis-cli --scan --pattern 'rl:tenant-42:*' | xargs redis-cli del
```

---

## Tips

- **TTL everything that's a cache** — Redis is memory-bound; un-expired keys are a slow leak.
- Set `maxmemory-policy allkeys-lru` for cache workloads so Redis evicts instead of OOMing.
- It's **single-threaded** — avoid O(N) commands (`KEYS`, big `LRANGE`) on the hot path; use `SCAN`, bounded ranges, and pipelining.
- Persistence is a choice: **RDB** snapshots (fast restart, some loss) vs **AOF** (durable, larger) — a pure cache can run with neither.
- Store vectors as raw `Float32Array` buffers; match `DIM`/`DISTANCE_METRIC` to your embedding model (cosine for normalized embeddings).
- Use Redis for hot/ephemeral state (cache, rate limits, locks, queues); reach for a dedicated vector DB ([turbopuffer.md](turbopuffer.md)) when the corpus outgrows memory.
- Reuse one client at module scope; `duplicate()` only for blocking pub/sub subscribers.
