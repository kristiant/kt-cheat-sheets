# Agentic RAG

**What it is:** RAG where an LLM agent *drives* retrieval as a multi-step, looping decision — deciding whether to retrieve, what and where to search, grading what comes back, and retrying — instead of a fixed retrieve-once-then-generate pipeline.

**Why people use it:** One-shot RAG fails on multi-hop questions, ambiguous queries, and bad retrievals. An agent can re-query, switch sources, grade relevance, and self-correct — trading latency and cost for accuracy.

**Typically used for:** Complex/multi-hop Q&A, multi-source knowledge (docs + SQL + web), and high-stakes answers where "retrieved the wrong thing" is unacceptable.

> Classical one-shot RAG, chunking, embeddings, and retrieval techniques (dense/BM25/hybrid/rerank) are in [rag.md](rag.md). This sheet is the agentic layer on top. Examples use LangGraph — see [langgraph.md](langgraph.md), [agentic-patterns.md](../practices/agentic-patterns.md).

---

## Classical vs agentic

| | Classical (one-shot) | Agentic |
|---|---|---|
| Retrieval | always, once, fixed | a decision/tool the agent calls 0..N times |
| Query | user's query verbatim | rewritten, decomposed, routed |
| Bad results | passed to the model anyway | graded, then re-query / switch source / web fallback |
| Control flow | linear pipeline | graph with loops + conditional edges |
| Cost / latency | low | higher (multiple LLM calls) |

The core shift: **retrieval becomes a tool and a graded loop, not a pipeline step.**

---

## Building-block patterns

### Retrieval as a tool

Give the agent a `retrieve` tool and let it decide when to call it (and with what query). Trivial questions skip retrieval entirely.

```ts
import { tool } from "@langchain/core/tools";
import { createReactAgent } from "@langchain/langgraph/prebuilt";
import { z } from "zod";

const retrieveTool = tool(
  async ({ query }) => (await retriever.invoke(query)).map((d) => d.pageContent).join("\n\n"),
  { name: "retrieve", description: "Search the knowledge base", schema: z.object({ query: z.string() }) },
);

const agent = createReactAgent({ llm, tools: [retrieveTool] });
```

### Routing (multi-source)

Pick the right knowledge source before retrieving — vector store vs SQL vs web.

```ts
const Route = z.object({ source: z.enum(["docs", "sql", "web"]) });
const router = llm.withStructuredOutput(Route);
const { source } = await router.invoke(`Which source answers: ${question}?`);
```

### Query rewriting & decomposition

Rewrite a vague query, or split a multi-hop one into sub-questions retrieved independently.

```ts
const Plan = z.object({ subQuestions: z.array(z.string()) });
const planner = llm.withStructuredOutput(Plan);
const { subQuestions } = await planner.invoke(`Break into sub-questions: ${question}`);
// retrieve each sub-question, then synthesize
```

### Document grading (Corrective RAG)

Grade retrieved docs for relevance *before* generating. If they're weak, rewrite the query or fall back to web search.

```ts
const Grade = z.object({ relevant: z.boolean() });
const grader = llm.withStructuredOutput(Grade);

const graded = await Promise.all(
  docs.map(async (d) => ({ doc: d, ...(await grader.invoke(`Q: ${question}\nDoc: ${d.pageContent}\nRelevant?`)) })),
);
const kept = graded.filter((g) => g.relevant).map((g) => g.doc);
const needsFallback = kept.length === 0;
```

### Answer grading & self-correction (Self-RAG)

After generating, check the answer is **grounded** in the docs and **addresses** the question. If not, loop — retrieve more or regenerate.

```ts
const Check = z.object({ grounded: z.boolean(), addressesQuestion: z.boolean() });
const checker = llm.withStructuredOutput(Check);
const verdict = await checker.invoke(`Docs: ${context}\nAnswer: ${answer}\nGrounded & on-topic?`);
// !grounded → hallucination, regenerate ; !addressesQuestion → retrieve again
```

### Adaptive routing (by complexity)

Route simple queries to a direct answer (or single retrieval) and complex ones to the full agentic loop — don't pay the cost when you don't need it.

---

## Canonical LangGraph implementation (CRAG-style)

The grading loop wired as a graph: retrieve → grade → (generate | rewrite → web search) → generate.

```ts
import { Annotation, StateGraph, START, END } from "@langchain/langgraph";

const State = Annotation.Root({
  question: Annotation<string>,
  documents: Annotation<string[]>({ reducer: (_, b) => b, default: () => [] }),
  generation: Annotation<string>,
  tries: Annotation<number>({ reducer: (_, b) => b, default: () => 0 }),
});

const graph = new StateGraph(State)
  .addNode("retrieve", retrieveNode)
  .addNode("gradeDocs", gradeDocsNode)       // filters docs, sets a "relevant?" signal
  .addNode("rewrite", rewriteQueryNode)      // improves the question
  .addNode("webSearch", webSearchNode)       // fallback source
  .addNode("generate", generateNode)
  .addEdge(START, "retrieve")
  .addEdge("retrieve", "gradeDocs")
  .addConditionalEdges("gradeDocs", decideAfterGrading, {
    generate: "generate",                    // docs are good
    rewrite: "rewrite",                      // docs weak → improve query
  })
  .addEdge("rewrite", "webSearch")
  .addEdge("webSearch", "generate")
  .addConditionalEdges("generate", checkAnswer, {
    done: END,                               // grounded + on-topic
    retry: "rewrite",                        // hallucinated/off-topic → loop (with a cap)
  })
  .compile();
```

```ts
// route function — boring, returns a key; cap loops to avoid infinite retrieval
function decideAfterGrading(s: typeof State.State) {
  return s.documents.length > 0 ? "generate" : "rewrite";
}
function checkAnswer(s: typeof State.State) {
  return s.tries >= 2 ? "done" : (isGrounded(s) ? "done" : "retry");
}
```

> Always cap the loop (`tries`, or `recursionLimit` on invoke) — a grader that never passes will retrieve forever. See [agentic-patterns.md](../practices/agentic-patterns.md).

---

## Multi-agent RAG

For broad systems, split into agents: a **router** delegates to domain-specific retriever agents (legal, billing, product), each with its own index, and a **synthesizer** merges answers. Use a supervisor (see [agentic-patterns.md](../practices/agentic-patterns.md)). Justified when sources are genuinely separate; otherwise one graded loop is simpler.

---

## Named techniques

- **CRAG (Corrective RAG)** — a retrieval grader scores docs; correct → use, ambiguous → refine, incorrect → **web-search fallback**. The document-grading loop above.
- **Self-RAG** — the model decides *when* to retrieve and critiques its own output (grounded? relevant?) via reflection, looping until satisfied.
- **Adaptive RAG** — a classifier routes by query complexity (no-retrieval / single-shot / multi-step agent).

> **Grounded:** These are the established agentic-RAG patterns — see LangChain's [Self-Reflective RAG with LangGraph](https://www.langchain.com/blog/agentic-rag-with-langgraph) (reference implementations of CRAG, Self-RAG, Adaptive RAG) and the [Agentic RAG survey](https://github.com/asinghcsu/AgenticRAG-Survey).

---

## When to use which

| Use **classical** RAG | Use **agentic** RAG |
|---|---|
| Simple factual lookup | Multi-hop / comparative questions |
| One knowledge source | Multiple sources to route between |
| Latency/cost sensitive | Accuracy worth extra LLM calls |
| Retrieval is reliably good | Retrieval often misses → needs grading/retry |

Start classical. Add agentic steps **only where they fix an observed failure** — grading when retrieval is noisy, rewriting when queries are vague, multi-hop when questions span documents. Each step adds latency and cost.

---

## Practical recipes

**Minimal agentic RAG (retrieval as a tool):**
```ts
const agent = createReactAgent({ llm, tools: [retrieveTool] });
await agent.invoke({ messages: [{ role: "user", content: question }] });
```

**Grade docs, drop irrelevant ones before generating:**
```ts
const kept = (await Promise.all(docs.map(gradeRelevance))).filter((g) => g.relevant).map((g) => g.doc);
```

**Web-search fallback when retrieval is empty/weak:**
```ts
const context = kept.length ? kept : await webSearch(question);
```

**Cap the retrieval loop (avoid infinite retries):**
```ts
await graph.invoke({ question }, { recursionLimit: 10 });
```

**Decompose a multi-hop question:**
```ts
const { subQuestions } = await planner.invoke(question);
const all = (await Promise.all(subQuestions.map((q) => retriever.invoke(q)))).flat();
```

---

## Failure modes & fixes

| Symptom | Cause | Fix |
|---|---|---|
| Loops forever / `GraphRecursionError` | grader never passes | cap with `tries`/`recursionLimit` + a "good enough" exit |
| Slow & expensive | retrieving/grading every turn | adaptive routing — skip the loop for simple queries |
| Still hallucinating | no answer-grounding check | add a Self-RAG grounded/on-topic gate before returning |
| Right docs, wrong source | no routing | route to the correct index/tool first |
| Rewrites drift off-topic | unconstrained query rewriting | keep the original question in state; rewrite from it, not the last rewrite |

## Tips

- Retrieval is a **tool and a graded loop**, not a fixed step — that's the whole shift from classical RAG.
- Cap every loop (`recursionLimit`, a `tries` counter) — graders that never pass are the #1 agentic-RAG bug.
- Grade documents (relevance) *and* the answer (grounding) — they catch different failures (CRAG vs Self-RAG).
- Add agentic steps to fix a measured failure, not preemptively; each one costs latency and money.
- Keep route functions boring — return a node name or `END`, decide on graded state.
- Reuse the classical retrieval stack ([rag.md](rag.md)) underneath; agentic RAG orchestrates it, it doesn't replace it.
