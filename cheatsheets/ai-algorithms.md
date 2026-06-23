# Algorithms Behind AI Systems

**What it is:** The core algorithms an AI engineer actually meets — similarity search, clustering, graph traversal, and ranking — each framed by *where it shows up* in real AI systems.

**Why people use it:** RAG, agents, and vector DBs are built on these. Knowing how they work (and their tradeoffs) is the difference between configuring a vector index blindly and tuning it on purpose.

**Typically used for:** Understanding vector search internals, clustering embeddings, traversing knowledge/agent graphs, and ranking results.

> Pipeline-level retrieval usage is in [rag.md](rag.md); this sheet is the *algorithms*. Each entry: what it is · where it shows up in AI · the key tradeoff.

---

## Similarity & nearest neighbour

### Distance metrics

How "close" two vectors are. **cosine** (angle, ignores magnitude — default for normalized embeddings) · **dot product** (cosine × magnitudes) · **L2 / euclidean** (straight-line distance).

*Where:* every embedding comparison. Pick the one your embedding model was trained for (usually cosine). See [rag.md](rag.md).

### k-NN (exact) vs ANN (approximate)

- **k-NN** — compare the query to *every* vector, return the k closest. Exact but **O(n·d)** per query — too slow past ~10⁴–10⁵ vectors.
- **ANN** — trade a little recall for huge speed; sub-linear search over millions of vectors. What every vector DB actually runs.

*Tradeoff:* ANN recall < 100% — you may miss a true neighbour. Tune index params for the recall/latency point you need.

### ANN index types

| Index | How it works | Trade |
|---|---|---|
| **HNSW** | multi-layer "small-world" graph; greedy-descend from a sparse top layer to dense bottom. ~O(log n) search | fast + high recall, but high memory; the default |
| **IVF** | **k-means** partitions vectors into `nlist` cells; query searches the `nprobe` nearest cells | smaller memory; recall depends on `nprobe` |
| **PQ** (product quantization) | split each vector into sub-vectors, quantize each to a codebook → store compressed codes | massive memory savings, lossy; often layered on IVF (IVF-PQ) |

*Where:* this is the engine inside pgvector, Pinecone, Qdrant, turbopuffer ([turbopuffer.md](turbopuffer.md)).

### Top-k selection

Keep the k best from a stream without sorting everything: a **min-heap of size k** → **O(n log k)**, not O(n log n).

*Where:* retrieving the top-k chunks *and* top-k token decoding — same primitive, two layers of the stack.

---

## Clustering

### k-means

Partition n points into k clusters:

```
1. init k centroids (k-means++ picks spread-out starts)
2. assign each point to its nearest centroid
3. recompute each centroid = mean of its points
4. repeat 2–3 until assignments stop changing
```

**O(n·k·i·d)**. Pick k with the elbow method or silhouette score.

*Where:* building **IVF index cells**, clustering embeddings for topic discovery, near-duplicate detection, dataset exploration.
*Tradeoff:* you must choose k upfront; assumes round, similar-size clusters; sensitive to init (hence k-means++).

### When k-means fails → DBSCAN / hierarchical

- **DBSCAN** — density-based; finds arbitrary shapes, needs no k, labels outliers as noise (params: `eps`, `minPts`).
- **Hierarchical** — builds a cluster tree (dendrogram); cut it at any level, no fixed k.

*Use when:* clusters aren't spherical, k is unknown, or you need outlier detection.

---

## Graph & search

### BFS vs DFS

Both traverse a graph in **O(V + E)**; the data structure differs:

| | BFS | DFS |
|---|---|---|
| Frontier | **queue** (FIFO) | **stack** / recursion |
| Order | level by level | deep down one path first |
| Best for | shortest path in *unweighted* graphs, "nearest first" | cycle detection, topological sort, exhaustive exploration |

*Where:* GraphRAG entity traversal, dependency/order resolution, crawling, and reasoning over a graph. Note **LangGraph is a graph** ([langgraph-core.md](langgraph-core.md)) and tree-of-thought / agent search are BFS/DFS over reasoning states.

### Dijkstra / A*

Shortest path in a **weighted** graph (non-negative edges) via a priority queue — **O((V+E) log V)**. **A*** = Dijkstra + a heuristic that steers toward the goal (faster when you can estimate remaining cost).

*Where:* planning, routing, cost-aware agent/tool selection.

### Beam search

Keep the **B** highest-scoring partial sequences at each step (not just the single best like greedy, not all like exhaustive). Width B trades quality for compute.

*Where:* LLM **decoding** — deterministic tasks (translation, structured generation) more than open-ended chat. See sampling (top-k/top-p/temperature) in [rag.md](rag.md) / [agentic-patterns.md](agentic-patterns.md).

---

## Ranking & fusion

- **TF-IDF / BM25** — lexical relevance scoring (term frequency × rarity). The sparse half of hybrid search.
- **Reciprocal Rank Fusion (RRF)** — merge multiple ranked lists by rank, not score: `score(d) = Σ 1/(k + rank_i(d))` (k≈60). How dense + BM25 results combine without comparable scores.
- **Cross-encoder rerank** — re-score query+doc *pairs* jointly for precision; slower, so applied to a top-N over-fetch.

*Where:* the retrieve → fuse → rerank pipeline. Mechanics here; usage in [rag.md](rag.md).

---

## Complexity at a glance

| Operation | Cost |
|---|---|
| Exact k-NN (brute force) | O(n·d) per query |
| ANN (HNSW) search | ~O(log n) |
| k-means | O(n·k·i·d) |
| BFS / DFS | O(V + E) |
| Dijkstra | O((V+E) log V) |
| Top-k via heap | O(n log k) |

---

## Tips

- Default to **cosine + HNSW** for vector search; reach for IVF/PQ only when memory forces it.
- ANN is approximate — if recall matters, measure it (over-fetch + rerank covers most gaps).
- **k-means is everywhere** under the hood (IVF cells, dedup) — recognise it even when you're not calling it directly.
- BFS = shortest-path/nearest-first; DFS = exhaustive/topological. Cap depth when searching reasoning graphs or you won't terminate.
- Top-k via a heap, not a full sort, when n is large and k is small.
- These are the *primitives*; the [rag.md](rag.md) / [rag-agentic.md](rag-agentic.md) sheets show them composed into pipelines.
