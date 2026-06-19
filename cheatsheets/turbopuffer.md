# turbopuffer

**What it is:** A serverless vector + full-text search database built on object storage (S3/GCS) — you store rows of vectors and attributes, then query by vector similarity (ANN), BM25 full-text, or both, with filters.

**Why people use it:** Storage-on-object-store makes it far cheaper than memory-resident vector DBs, scales to billions of vectors, and is serverless (no clusters to size). Vector *and* keyword search in one query.

**Typically used for:** RAG retrieval, semantic + keyword search, recommendations, and any "find similar / find matching" over large corpora.

> The retrieval techniques (dense/BM25/hybrid/rerank) are in [rag.md](rag.md); this sheet is the turbopuffer API. TS SDK: `@turbopuffer/turbopuffer`.

---

## Mental model

```
namespace            a collection (like a table) — created on first write
  └─ rows            { id, vector?, ...attributes }
       query by   rank_by  →  vector ANN  or  BM25 full-text
       narrow by  filters  →  [field, op, value]
       indexed by schema   →  which attributes are filterable / full-text
```

A row is an `id` + an optional `vector` + arbitrary attributes. You query with `rank_by` (how to score) and `filters` (what to include).

---

## Setup

```bash
npm i @turbopuffer/turbopuffer
```

```ts
import Turbopuffer from "@turbopuffer/turbopuffer";

const tpuf = new Turbopuffer({
  apiKey: process.env.TURBOPUFFER_API_KEY,
  region: "aws-us-east-1",          // your namespace's region
});

const ns = tpuf.namespace("documents");
```

---

## Write (upsert)

Namespaces are created on first write. A row is an `id` + a `vector` (the embedding) + **attributes — the row's real content and metadata**. A typical RAG row stores the chunk text *and* its embedding:

```ts
await ns.write({
  distance_metric: "cosine_distance",            // or "euclidean_squared" — set on first write
  upsert_rows: [
    {
      id: "policy.pdf#chunk-0",
      vector: await embed("Refunds are issued within 5 business days of approval."),
      text: "Refunds are issued within 5 business days of approval.",   // the real content
      source: "policy.pdf",
      page: 3,
    },
  ],
  schema: {
    text:   { type: "string", full_text_search: true },   // BM25 over the stored text
    source: { type: "string", filterable: true },
  },
});
```

> The **vector** is the embedding (for semantic ANN); the **attributes are the actual data** you store, filter on, and read back. **BM25 searches a string attribute (`text`), not the vector** — so you must store the real text to keyword-search or display it. Store both on the same row and you get hybrid search.

- `id` is `string | number`. `vector` length must match across the namespace.
- Re-writing an existing `id` upserts (replaces) it.

Delete and patch in the same `write` call:

```ts
await ns.write({ deletes: [1, 2] });                          // by id
await ns.write({ delete_by_filter: ["category", "Eq", "spam"] });
await ns.write({ patch_rows: [{ id: 3, category: "archived" }] }); // partial update
```

---

## Query

### Vector search (ANN)

```ts
const res = await ns.query({
  rank_by: ["vector", "ANN", await embed("how long do refunds take?")],
  top_k: 10,
  include_attributes: ["text", "source"],        // or true for all
  filters: ["source", "Eq", "policy.pdf"],
});
res.rows;   // [{ id, $dist, text, source }, ...]
```

### Full-text search (BM25)

```ts
const res = await ns.query({
  rank_by: ["text", "BM25", "refund window"],    // [attribute, "BM25", query]
  top_k: 10,
  filters: ["source", "Eq", "policy.pdf"],
});
```

The attribute must be `full_text_search`-enabled in the schema (below).

---

## Filters

`[field, op, value]`, combined with `And` / `Or`:

```ts
filters: ["age", "Gt", 18]

filters: ["And", [
  ["category", "Eq", "billing"],
  ["year", "Gte", 2024],
  ["tag", "In", ["urgent", "vip"]],
]]
```

Operators: `Eq`, `NotEq`, `Gt`, `Gte`, `Lt`, `Lte`, `In`, `NotIn`, `Glob` / `NotGlob` (wildcards), `Contains` (array membership). Only **filterable** attributes (per schema) can be filtered.

---

## Schema

Attribute indexing is inferred on write, but set it explicitly to control filtering and full-text:

```ts
await ns.write({
  upsert_rows: [/* ... */],
  schema: {
    title:    { type: "string", full_text_search: true },  // enable BM25
    category: { type: "string", filterable: true },
    year:     { type: "uint", filterable: true },
  },
});
```

- `full_text_search: true` enables BM25 on a `string` / `[]string` attribute (not filterable by default — set `filterable: true` to also filter it).
- `filterable: true` is required to use an attribute in `filters`.
- Inspect the live schema: `await ns.schema()`.

---

## Other operations

```ts
await tpuf.namespaces({ prefix: "doc" });   // list namespaces
await ns.metadata();                        // row count, approx size
await ns.schema();                          // current attribute schema
await ns.deleteAll();                       // ⚠️ delete the whole namespace
await ns.hintCacheWarm();                   // pre-warm cache before a burst of queries
await ns.recall({ top_k: 10, num: 100 });   // measure ANN recall vs exact
```

---

## Practical recipes

**Upsert embeddings for RAG:**
```ts
await ns.write({
  distance_metric: "cosine_distance",
  upsert_rows: chunks.map((c, i) => ({ id: c.id, vector: c.embedding, text: c.text, source: c.source })),
  schema: { text: { type: "string", full_text_search: true }, source: { type: "string", filterable: true } },
});
```

**Hybrid: run vector and BM25, fuse client-side:**
```ts
const [dense, lexical] = await Promise.all([
  ns.query({ rank_by: ["vector", "ANN", qvec], top_k: 20 }),
  ns.query({ rank_by: ["text", "BM25", queryText], top_k: 20 }),
]);
// merge by reciprocal-rank fusion (see rag.md)
```

**Multi-tenant: filter by tenant before search:**
```ts
await ns.query({ rank_by: ["vector", "ANN", qvec], top_k: 10, filters: ["tenantId", "Eq", tenantId] });
```

**Delete a tenant's data:**
```ts
await ns.write({ delete_by_filter: ["tenantId", "Eq", tenantId] });
```

**List + clean up namespaces:**
```ts
for (const n of await tpuf.namespaces({ prefix: "tmp-" })) await tpuf.namespace(n.id).deleteAll();
```

---

## Tips

- `distance_metric` is fixed at namespace creation (first write) — choose `cosine_distance` for normalized embeddings.
- An attribute must be `filterable` to appear in `filters`, and `full_text_search` to use BM25 — set the schema explicitly rather than relying on inference.
- Over-fetch then rerank: pull `top_k: 20–50` and rerank (see [rag.md](rag.md)); ANN top-k is recall-approximate.
- Always filter by tenant before vector search in multi-tenant apps — pre-filtering is cheaper and prevents leakage.
- `hintCacheWarm()` before a known query burst; cold namespaces hydrate from object storage on first hit (the cost of the cheap-storage model).
- Keep vector dimensions consistent within a namespace; mixed dimensions are rejected.
