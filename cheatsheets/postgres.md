# Postgres (for AI engineering)

**What it is:** An open-source relational database with strong SQL, transactions, JSON, full-text search, and — via [`pgvector`](https://github.com/pgvector/pgvector) — native vector similarity search.

**Why people use it:** One battle-tested store for relational data *and* embeddings, so RAG, metadata filters, and app data live in the same place with the same transactions — no second vector DB to sync.

**Typically used for:** Vector search for RAG, hybrid (vector + keyword + metadata) retrieval, storing chat history / agent state, JSONB document storage, and as the operational DB behind an AI app.

> TS-first ([`pg`](https://node-postgres.com), the `node-postgres` driver). Pure-vector-DB alternative: [turbopuffer.md](turbopuffer.md). Where these patterns sit in a pipeline: [rag.md](rag.md).

---

## Mental model

Tables of typed rows, queried with SQL, wrapped in **ACID transactions**. Connections are expensive and limited — you **pool** them. For AI work, the leverage is in three extensions/features layered on top: `pgvector` (embeddings), `JSONB` (schemaless metadata), and full-text search (`tsvector`).

```bash
npm i pg                 # node-postgres driver
npm i -D @types/pg
```

```ts
import { Pool } from "pg";

const pool = new Pool({ connectionString: process.env.DATABASE_URL });

const { rows } = await pool.query("select 1 + 1 as sum");
console.log(rows[0].sum);   // 2
```

---

## Fundamentals (node-postgres)

**Always use a `Pool`, never a bare `Client` per request.** A pool keeps a small set of connections open and hands them out; opening a new connection per query exhausts Postgres' `max_connections` fast.

```ts
const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  max: 10,                      // pool size — keep well under server max_connections
  idleTimeoutMillis: 30_000,    // close idle clients
  connectionTimeoutMillis: 5_000,
});
// reuse `pool` for the app's lifetime; pool.query() checks out + releases automatically
await pool.end();               // graceful shutdown
```

**Always parameterize — never string-concatenate values.** `$1`, `$2` placeholders are sent separately from the SQL, which prevents injection and lets Postgres cache the plan.

```ts
// ✅ parameterized
await pool.query("select * from docs where tenant_id = $1 and year = $2", [tenantId, 2024]);

// ❌ never — injectable, no plan caching
await pool.query(`select * from docs where tenant_id = '${tenantId}'`);
```

**Transactions need one checked-out client** (a pool can't guarantee the same connection across queries otherwise):

```ts
const client = await pool.connect();
try {
  await client.query("begin");
  await client.query("insert into runs(id) values ($1)", [runId]);
  await client.query("update budget set spent = spent + $1 where tenant = $2", [cost, tenant]);
  await client.query("commit");
} catch (e) {
  await client.query("rollback");
  throw e;
} finally {
  client.release();             // ALWAYS release back to the pool
}
```

**Serverless / Lambda:** a process-per-request model blows up pooling. Use a server-side pooler — [PgBouncer](https://www.pgbouncer.org) or [Neon](https://neon.tech)/[Supabase](https://supabase.com) pooled endpoints in **transaction mode** — and a small in-process pool (`max: 1–2`).

---

## Schema basics

```sql
create table documents (
  id          bigint generated always as identity primary key,
  tenant_id   text not null,
  source      text not null,
  content     text not null,
  metadata    jsonb not null default '{}',
  created_at  timestamptz not null default now()
);

create index on documents (tenant_id);              -- filter by tenant
create index on documents using gin (metadata);      -- query inside JSONB
```

| Type | Use for |
|---|---|
| `text` | strings (no length penalty vs `varchar` in PG) |
| `jsonb` | schemaless metadata, tool args, LLM outputs — indexable |
| `timestamptz` | always store time **with** zone (UTC) |
| `bigint … identity` | modern auto-increment PK (over `serial`) |
| `uuid` | distributed/exposed IDs (`gen_random_uuid()`) |
| `vector(N)` | embeddings (pgvector) |

---

## pgvector — embeddings & similarity search

The reason Postgres shows up in RAG. Stores embedding vectors and finds nearest neighbours by distance.

```sql
create extension if not exists vector;

alter table documents add column embedding vector(1536);   -- match your model's dims
```

**Distance operators** — pick the one matching how the embedding model was trained:

| Operator | Distance | Use when |
|---|---|---|
| `<=>` | cosine | **default** for OpenAI / most embeddings (normalized) |
| `<#>` | negative inner product | normalized vectors, slightly faster |
| `<->` | L2 / Euclidean | when the model is trained for it |

```sql
-- k nearest neighbours to a query embedding
select id, content, 1 - (embedding <=> $1) as similarity
from documents
where tenant_id = $2
order by embedding <=> $1
limit 5;
```

Insert from TS — pgvector takes a **string-formatted array**:

```ts
const vec = `[${embedding.join(",")}]`;   // "[0.1,0.2,...]"
await pool.query(
  "insert into documents (tenant_id, content, embedding) values ($1, $2, $3)",
  [tenantId, content, vec],
);
```

> Or `npm i pgvector` and `import pgvector from "pgvector/pg"` + `pgvector.registerType(pool)` to pass/receive real arrays instead of hand-formatting.

### ANN index — HNSW

Without an index, KNN is an exact **sequential scan** (fine for <10k rows, slow beyond). Add an approximate index for speed:

```sql
create index on documents using hnsw (embedding vector_cosine_ops);
-- match the ops class to your operator: vector_cosine_ops ↔ <=>, vector_ip_ops ↔ <#>, vector_l2_ops ↔ <->
```

```sql
set hnsw.ef_search = 100;   -- higher = better recall, slower (per-session tunable)
```

> **HNSW vs IVFFlat:** HNSW builds slower and uses more memory but gives better recall/speed and needs no training data — the production default. IVFFlat is lighter but must be built *after* data exists and needs `lists` tuned. Prefer HNSW unless memory-bound.

> ⚠️ ANN indexes return **approximate** results, and a filtered `where` can interact badly with the index (it may scan past the index limit). For strict tenant isolation, test recall, or use [partitioning](#partitioning) / partial indexes.

---

## Graph queries — SQL/PGQ (Postgres 19)

Relational/multi-hop questions ("which customers were hit by the outage caused by the dependency team X owns?") are graph traversals that plain `JOIN`s express awkwardly. **Postgres 19** adds [`SQL/PGQ`](https://www.postgresql.org/about/news/postgresql-19-beta-1-released-3313/) (the SQL:2023 property-graph standard) — graph pattern-matching over your *existing* tables, no new storage engine and no extension.

> ⚠️ **Status:** Postgres 19 is in **Beta** (Beta 1, June 2026; GA later in 2026). Beta 1 supports **fixed-depth** patterns only — arbitrary variable-length paths (`-[:edge]->{1,5}`) are slated for a later release. For deep, unbounded traversal today, a dedicated graph DB still wins.

A **property graph** is metadata mapping tables → vertices/edges; you then query it with `GRAPH_TABLE`:

```sql
create property graph deps
  vertex tables (
    teams       key (id) label team       properties (name),
    services    key (id) label service     properties (name),
    customers   key (id) label customer    properties (name)
  )
  edge tables (
    owns      source key (team_id) references teams (id)
              destination key (service_id) references services (id) label owns,
    uses      source key (customer_id) references customers (id)
              destination key (service_id) references services (id) label uses
  );

-- which customers use a service owned by team 'platform'?
select customer_name from graph_table (deps
  match (t:team)-[:owns]->(s:service)<-[:uses]-(c:customer)
  where t.name = 'platform'
  columns (c.name as customer_name)
);
```

The pattern is rewritten into ordinary joins — same planner, same indexes. It's a more readable way to express relational hops than nested `JOIN`s, in the same DB as your vectors and metadata.

> See [rag.md](rag.md#what-vector-search-cant-do-temporal--relational-reasoning) for why dense retrieval can't do this on its own.

---

## Hybrid search (vector + keyword)

Dense vectors miss exact terms (IDs, codes, names); full-text catches them. Postgres does both, so you can fuse them in one query — no second system.

```sql
alter table documents add column ts tsvector
  generated always as (to_tsvector('english', content)) stored;
create index on documents using gin (ts);
```

Fuse with **Reciprocal Rank Fusion** in a single statement:

```sql
with vec as (
  select id, row_number() over (order by embedding <=> $1) as rank
  from documents where tenant_id = $3 order by embedding <=> $1 limit 50
),
kw as (
  select id, row_number() over (order by ts_rank(ts, plainto_tsquery('english', $2)) desc) as rank
  from documents where tenant_id = $3 and ts @@ plainto_tsquery('english', $2) limit 50
)
select coalesce(vec.id, kw.id) as id,
       coalesce(1.0/(60 + vec.rank), 0) + coalesce(1.0/(60 + kw.rank), 0) as score
from vec full outer join kw using (id)
order by score desc
limit 10;
```

> Same RRF idea as [rag.md](rag.md#hybrid-retrieval), done in SQL. The `60` is the standard RRF `k` constant. Metadata filters (`where metadata->>'product' = ...`) drop straight into both CTEs.

---

## JSONB — schemaless metadata

For LLM outputs, tool arguments, and per-document metadata whose shape varies. Indexable and queryable, unlike a plain `text` blob.

```sql
-- query into JSON
select * from documents where metadata->>'product' = 'portal';     -- ->> = text
select * from documents where metadata @> '{"tier": "pro"}';        -- @> = contains (uses GIN index)
select * from documents where (metadata->>'year')::int = 2024;      -- cast for numeric compare
```

```ts
await pool.query(
  "insert into documents (tenant_id, content, metadata) values ($1, $2, $3)",
  [tenantId, content, { product: "portal", year: 2024, tags: ["auth"] }],   // pg serializes objects to jsonb
);
```

> Use the `@>` containment operator (GIN-indexed) for filters; `->>'key' = ...` works but won't use the GIN index as well. Index a hot scalar key directly: `create index on documents ((metadata->>'product'));`

---

## Chat history / agent state

A common pattern: thread-scoped message log, append-only, read back in order.

```sql
create table messages (
  id         bigint generated always as identity primary key,
  thread_id  text not null,
  role       text not null check (role in ('system','user','assistant','tool')),
  content    text not null,
  metadata   jsonb not null default '{}',
  created_at timestamptz not null default now()
);
create index on messages (thread_id, created_at);
```

```ts
async function appendMessage(threadId: string, role: string, content: string) {
  await pool.query(
    "insert into messages (thread_id, role, content) values ($1, $2, $3)",
    [threadId, role, content],
  );
}

async function loadHistory(threadId: string, limit = 50) {
  const { rows } = await pool.query(
    "select role, content from messages where thread_id = $1 order by created_at desc limit $2",
    [threadId, limit],
  );
  return rows.reverse();   // chronological for the prompt
}
```

> [LangGraph](langgraph.md)'s `PostgresSaver` checkpointer persists graph state to Postgres out of the box — same idea, managed.

---

## Concurrency & idempotency

**Atomic upsert** — insert or update on conflict (e.g. caching an embedding, dedup by hash):

```sql
insert into cache (key, value) values ($1, $2)
on conflict (key) do update set value = excluded.value, updated_at = now();
```

**Idempotency key** — make a write safe to retry (webhooks, agent steps):

```sql
insert into runs (idempotency_key, payload) values ($1, $2)
on conflict (idempotency_key) do nothing
returning id;                 -- empty result ⇒ already processed, skip
```

**Job queue with `for update skip locked`** — Postgres as a simple, transactional worker queue without Redis:

```sql
select id, payload from jobs
where status = 'pending'
order by created_at
for update skip locked          -- each worker grabs a different row
limit 1;
-- then: update jobs set status = 'done' where id = $1; commit;
```

> `skip locked` is how Postgres-backed queues (e.g. [graphile-worker](https://github.com/graphile/worker), River) let many workers pull jobs without colliding.

---

## Operating it

### Explain a slow query

```sql
explain (analyze, buffers) select ...;     -- shows the real plan + timing
```

Look for `Seq Scan` on big tables (missing index) and bad row estimates (run `analyze`).

### Partitioning

For huge append tables (events, messages), partition by time so old data is cheap to drop and queries hit fewer rows:

```sql
create table events (id bigint, created_at timestamptz, ...) partition by range (created_at);
create table events_2026_06 partition of events
  for values from ('2026-06-01') to ('2026-07-01');
```

### Connection budget

`max_connections` is finite (often ~100). `app instances × pool.max` must stay under it — this is the #1 production outage cause. Use a pooler (PgBouncer) when you have many app processes.

---

## Practical recipes

**Reset a dev database (destructive):**
```sql
drop schema public cascade; create schema public;
```
> ⚠️ Wipes everything in the schema. Dev only.

**Find the biggest tables:**
```sql
select relname, pg_size_pretty(pg_total_relation_size(relid)) as size
from pg_catalog.pg_statio_user_tables order by pg_total_relation_size(relid) desc limit 10;
```

**Kill long-running / stuck queries:**
```sql
select pid, now() - query_start as runtime, query from pg_stat_activity
where state = 'active' and now() - query_start > interval '5 minutes' order by runtime desc;
-- then: select pg_terminate_backend(<pid>);
```

**See what's blocking:**
```sql
select * from pg_locks where not granted;
```

**Add pgvector + an HNSW index to an existing table:**
```sql
create extension if not exists vector;
alter table documents add column embedding vector(1536);
create index on documents using hnsw (embedding vector_cosine_ops);
```

**Local Postgres with pgvector (Docker):**
```bash
docker run -d --name pg -p 5432:5432 -e POSTGRES_PASSWORD=pw pgvector/pgvector:pg17
```

---

## Failure modes & fixes

| Symptom | Likely cause | Fix |
|---|---|---|
| `too many connections` | pool-per-request or oversized pools | one `Pool`, low `max`, add PgBouncer |
| Vector search slow | no ANN index / seq scan | `create index … using hnsw …` |
| Poor vector recall after filtering | ANN + `where` interaction | tune `ef_search`, partial index, or test exact scan |
| Filters scan whole table | no index on filter column / wrong JSONB op | index the column; use `@>` with GIN |
| Wrong similarity ranking | operator ≠ embedding's distance | match `<=>`/`<#>`/`<->` to the model |
| Injection / no plan caching | string-concatenated SQL | parameterize with `$1, $2` |
| Connection leaks under load | `client` never `release()`d | release in `finally` |

## Tips

- **Pool once, parameterize always, release in `finally`** — the three rules that prevent most production incidents.
- Match the **distance operator to your embedding model**, and the index **ops class to the operator**.
- Keep `pool.max × instances < max_connections`; reach for PgBouncer before raising it.
- `JSONB` for variable metadata, real columns for anything you filter or join on hot paths.
- `explain (analyze, buffers)` before guessing — the planner tells you what's actually happening.
- One store for vectors + metadata + app data means **one transaction** — the main reason to keep RAG in Postgres instead of a separate vector DB.
