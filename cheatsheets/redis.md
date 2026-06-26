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
