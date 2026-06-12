# RAG (Retrieval-Augmented Generation)

**What it is:** A pattern that retrieves relevant text from your own data and injects it into an LLM's prompt, so answers are grounded in that data instead of the model's training.

**Why people use it:** Gives the model private/current knowledge without fine-tuning, cuts hallucination, and lets you cite sources. Cheaper and faster to update than retraining.

**Typically used for:** Q&A over docs, support bots, search over internal knowledge bases, and any "chat with your data" feature.

---

## The pipeline

```
INGEST → CHUNK → EMBED → INDEX        (offline, once per document)
                                  ↓
QUERY → [TRANSFORM] → RETRIEVE → RERANK → ASSEMBLE → GENERATE   (per request)
                                                          ↓
                                                        EVAL
```

Most quality problems trace to **chunking** and **retrieval**, not the LLM. Fix those first.

Examples use [LangChain.js](https://js.langchain.com). Install per section; common base:

```bash
npm i langchain @langchain/core @langchain/openai @langchain/community
```

---

## Foundations

### Chunking

Splitting documents into retrievable units. The single biggest quality lever — too big dilutes relevance, too small loses context.

```ts
import { RecursiveCharacterTextSplitter } from "@langchain/textsplitters";

const splitter = new RecursiveCharacterTextSplitter({
  chunkSize: 1000,      // chars (or tokens with a token splitter)
  chunkOverlap: 150,    // overlap keeps context across boundaries
});
const chunks = await splitter.splitDocuments(docs);
```

| Strategy | Splits on | Use when |
|---|---|---|
| Fixed-size | N chars/tokens | Uniform text, simplest |
| Recursive | paragraphs → lines → words | Default; respects structure |
| Semantic | embedding similarity shifts | Topic boundaries matter |
| Document-aware | Markdown headings, code blocks | Structured source (see docling `HybridChunker`) |

Rules of thumb: start at **~500–1000 tokens, ~10–20% overlap**. Smaller chunks = sharper retrieval but more fragments to assemble.

### Embeddings

A model maps text → a vector; similar meaning → nearby vectors (cosine similarity). Same model **must** embed both documents and queries.

```ts
import { OpenAIEmbeddings } from "@langchain/openai";

const embeddings = new OpenAIEmbeddings({ model: "text-embedding-3-small" });
const vec = await embeddings.embedQuery("how do I reset my password?");
```

### Vector store + ANN

Stores vectors and finds nearest neighbours fast via **approximate nearest neighbour** (ANN) indexes — HNSW (graph, default in pgvector/Qdrant) or IVF. Approximate = trades a little recall for big speed.

```ts
import { MemoryVectorStore } from "langchain/vectorstores/memory";

const store = await MemoryVectorStore.fromDocuments(chunks, embeddings);
// prod: pgvector, Pinecone, Qdrant, Weaviate — same interface
```

---

## Retrieval

### Dense vector lookup

Embed the query, return the nearest chunks. Good at *semantic* match ("car" ≈ "automobile"), weak on exact terms (codes, names, IDs).

```ts
const retriever = store.asRetriever({ k: 4 });
const docs = await retriever.invoke("how do I reset my password?");
```

### BM25 (sparse / lexical)

Classic keyword scoring (TF-IDF family). Strong on exact terms, acronyms, and rare tokens; no semantics. No embeddings needed.

```ts
import { BM25Retriever } from "@langchain/community/retrievers/bm25";

const bm25 = BM25Retriever.fromDocuments(chunks, { k: 4 });
const docs = await bm25.invoke("ERR_CONN_REFUSED reset token");
```

> **Lemmatization/stemming** (mapping "resets"/"resetting" → "reset") only matters for lexical search like BM25 — it raises keyword recall across inflections. Dense retrieval doesn't need it; embeddings already handle morphology. Production engines (Elasticsearch analyzers, Postgres `tsvector`) do it in config; LangChain's in-memory `BM25Retriever` does not.

### Hybrid retrieval

Run dense + BM25 and fuse with **Reciprocal Rank Fusion (RRF)** — covers both semantic and exact-term matches. The default for production retrieval.

```ts
import { EnsembleRetriever } from "langchain/retrievers/ensemble";

const hybrid = new EnsembleRetriever({
  retrievers: [bm25, store.asRetriever({ k: 4 })],
  weights: [0.4, 0.6],   // tune per corpus
});
const docs = await hybrid.invoke("password reset ERR_CONN_REFUSED");
```

> RRF scores each doc as `Σ 1/(k + rank)` across retrievers — rank-based, so it fuses without needing comparable scores.

---

## Query transformation

The user's raw question is often a poor search query. Rewrite it before retrieving.

### Query expansion / multi-query

LLM generates several paraphrases; retrieve each, union the results. Raises recall when wording mismatches the docs.

```ts
import { MultiQueryRetriever } from "langchain/retrievers/multi_query";
import { ChatOpenAI } from "@langchain/openai";

const mq = MultiQueryRetriever.fromLLM({
  llm: new ChatOpenAI({ model: "gpt-4o-mini" }),
  retriever: store.asRetriever(),
});
const docs = await mq.invoke("why won't my login work?");
```

### HyDE (Hypothetical Document Embeddings)

Ask the LLM to *draft an answer*, then embed that draft instead of the question — a fake answer is often closer in vector space to the real passage than the question is.

```ts
const draft = await llm.invoke(`Write a short passage answering: ${query}`);
const docs = await store.similaritySearch(draft.content, 4);
```

### Decomposition

Break a multi-hop question into sub-questions, retrieve for each, combine. For "Compare X's and Y's pricing" → retrieve X, retrieve Y separately.

---

## Structured & filtered retrieval

### Text-to-SQL

When the answer lives in a relational DB, not prose, have the LLM write SQL against your schema. "RAG over a database."

```ts
import { SqlDatabase } from "langchain/sql_db";
import { createSqlQueryChain } from "langchain/chains/sql_db";

const db = await SqlDatabase.fromDataSourceParams({ appDataSource: datasource });
const chain = await createSqlQueryChain({ llm, db, dialect: "postgres" });

const sql = await chain.invoke({ question: "How many orders shipped last week?" });
const result = await db.run(sql);
```

> ⚠️ Run generated SQL against a **read-only** role. Never let an LLM-authored query hit a writable connection.

### Metadata filtering (self-query)

Pre-filter by structured fields (date, tenant, source) *before* vector search — essential for multi-tenant and "only 2024 docs" cases. Self-query lets the LLM extract filters from natural language.

```ts
import { SelfQueryRetriever } from "langchain/retrievers/self_query";
import { FunctionalTranslator } from "@langchain/core/structured_query";

const retriever = SelfQueryRetriever.fromLLM({
  llm,
  vectorStore: store,
  documentContents: "Support articles",
  attributeInfo: [
    { name: "year", type: "number", description: "Year published" },
    { name: "product", type: "string", description: "Product name" },
  ],
  structuredQueryTranslator: new FunctionalTranslator(),
});
// "auth bugs in the portal from 2024" → filter {product: portal, year: 2024} + vector search
```

Plain metadata filter (no LLM) when you already know the filter:

```ts
await store.similaritySearch("auth bug", 4, { product: "portal", year: 2024 });
```

---

## Smarter indexing

### Parent-document (small-to-big)

Retrieve on **small** chunks (precise matching) but return their **larger parent** (full context) to the LLM. A common production default.

```ts
import { ParentDocumentRetriever } from "langchain/retrievers/parent_document";

const retriever = new ParentDocumentRetriever({
  vectorstore: store,                 // stores small child chunks
  byteStore,                          // stores parent docs
  childSplitter: new RecursiveCharacterTextSplitter({ chunkSize: 400 }),
  parentSplitter: new RecursiveCharacterTextSplitter({ chunkSize: 2000 }),
});
```

> **Sentence-window** is the same idea at finer grain: match on single sentences, return the few sentences around each hit.

### Contextual retrieval

Before embedding, prepend an LLM-generated blurb situating each chunk in its document ("This is from the 2024 refund policy, section 3…"). Sharply improves retrieval on chunks that are ambiguous out of context.

```ts
// per chunk, at index time:
const context = await llm.invoke(
  `Document: ${fullDoc}\nChunk: ${chunk}\nGive a 1-sentence context for this chunk.`
);
const enriched = `${context.content}\n\n${chunk}`;   // embed THIS
```

> **Grounded:** [Anthropic's Contextual Retrieval](https://www.anthropic.com/news/contextual-retrieval) reports ~35–49% fewer retrieval failures when combined with BM25 + rerank. Same idea as Docling's `contextualize()` — give each chunk its context back before embedding.

---

## Reranking

First-stage retrieval favours recall (grab ~20–50 candidates); a **reranker** (cross-encoder) then reorders by true relevance and keeps the top few. Big precision win for low cost.

```ts
import { CohereRerank } from "@langchain/cohere";
import { ContextualCompressionRetriever } from "langchain/retrievers/contextual_compression";

const reranked = new ContextualCompressionRetriever({
  baseCompressor: new CohereRerank({ model: "rerank-v3.5", topN: 4 }),
  baseRetriever: hybrid.withConfig({ k: 25 }),   // over-fetch, then rerank down
});
const docs = await reranked.invoke("password reset on mobile");
```

> **Grounded:** Cohere's [Rerank](https://docs.cohere.com/docs/rerank-on-langchain) is the common managed reranker; the canonical production stack is **hybrid retrieve → over-fetch → rerank → top-k**.

---

## Context assembly

Between retrieval and the prompt:

- **Dedup** near-identical chunks (overlap produces them).
- **Order matters** — models attend least to the middle of long contexts ("lost in the middle", [Liu et al. 2023](https://arxiv.org/abs/2307.03172)). Put the strongest chunks first and last.
- **Budget tokens** — cap context so the question + instructions aren't crowded out.

---

## Generation

Inject context, instruct the model to answer *only* from it, and cite.

```ts
const prompt = `Answer using ONLY the context. If the answer isn't there, say you don't know.
Cite sources as [n].

Context:
${docs.map((d, i) => `[${i + 1}] ${d.pageContent}`).join("\n\n")}

Question: ${question}`;

const answer = await llm.invoke(prompt);
```

Key anti-hallucination moves: "only from context", an explicit "I don't know" escape hatch, and citations the user can verify.

---

## Evaluation

You can't improve what you don't measure. The standard metrics:

- **Faithfulness** — is the answer supported by the retrieved context? (catches hallucination)
- **Answer relevance** — does it address the question?
- **Context precision/recall** — did retrieval surface the right chunks?

[Ragas](https://docs.ragas.io) is the de-facto tool — **Python only**, so eval is the one stage that leaves a TS stack:

```python
from ragas import evaluate
from ragas.metrics import faithfulness, answer_relevancy, context_precision

scores = evaluate(dataset, metrics=[faithfulness, answer_relevancy, context_precision])
```

---

## Practical recipes

**Minimal RAG (dense only):**
```ts
const store = await MemoryVectorStore.fromDocuments(chunks, embeddings);
const docs = await store.asRetriever({ k: 4 }).invoke(question);
```

**Production retrieval stack (hybrid → rerank):**
```ts
const hybrid = new EnsembleRetriever({
  retrievers: [BM25Retriever.fromDocuments(chunks, { k: 25 }),
               store.asRetriever({ k: 25 })],
  weights: [0.4, 0.6],
});
const final = new ContextualCompressionRetriever({
  baseCompressor: new CohereRerank({ model: "rerank-v3.5", topN: 5 }),
  baseRetriever: hybrid,
});
```

**Raise recall on vague queries (multi-query):**
```ts
const docs = await MultiQueryRetriever
  .fromLLM({ llm, retriever: store.asRetriever() })
  .invoke(question);
```

**Multi-tenant isolation (always filter first):**
```ts
await store.similaritySearch(query, 4, { tenantId });
```

---

## Failure modes & fixes

| Symptom | Likely cause | Fix |
|---|---|---|
| Right doc never retrieved | chunks too big / wrong split | smaller chunks, semantic or document-aware splitting |
| Misses exact terms (IDs, codes) | dense-only retrieval | add BM25 → hybrid |
| Retrieves loosely-related junk | no reranking | over-fetch + rerank |
| Answer ignores good context | lost-in-the-middle / too much context | reorder, trim, fewer chunks |
| Confident wrong answers | weak prompt grounding | "only from context" + "I don't know" + citations |
| Cross-tenant leakage | no metadata filter | filter by tenant before vector search |

## Tips

- Tune **retrieval before the prompt** — most "the LLM is dumb" issues are retrieval issues.
- Embed queries and documents with the **same model**; never mix.
- Over-fetch then rerank beats trusting first-stage top-k.
- Store source/page/chunk-id metadata at index time — needed for filtering and citations.
- Re-embed the whole corpus when you change embedding models.
