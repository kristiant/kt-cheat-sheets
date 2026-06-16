# State of AI Engineering

Date: 2026-06-15
Scope: sampled current AI Engineer / Applied AI Engineer / AI Product Engineer / ML Engineer postings across Australia, the US, and the UK.
Method: treat postings as telemetry; record only concrete technical signals named in the ads or search snippets visible on 2026-06-15.

## Week-over-Week Change

### New or clearer this week

| Signal | Evidence | Notes |
| --- | --- | --- |
| `Google ADK` | Karbon AU, Curie US, loveholidays UK, M-KOPA snippet | Moved from edge mention to recurring agent-framework option. |
| `PydanticAI` | Titan AI US, Level US, Klue snippet, Improbable snippet | Clearer than last run; now visible alongside `LangGraph` rather than as a niche Python-only choice. |
| `Mastra` | Level US, Percepta UK, Centralize snippet | Real but still niche; shows up in product-agent roles, not enterprise baseline roles. |
| `Promptfoo` in AU | Catapult Sports AU | Important because last run had `Promptfoo` mainly as general market noise; this week it appears in a named Australian role. |
| Graph-oriented retrieval | Catapult Sports AU: `Neo4j`, `GraphRAG`, community detection | Stronger than last week's generic RAG/vector language. |
| TypeScript-first AI product stack | Bain AU: `TypeScript`, `Node.js`, `LangGraph`, `FastAPI`, `tRPC`, `Zod`, `Vitest`, `Vite`, `Turbopack`, `esbuild` | Stronger and more explicit than prior TS/Node examples. |
| `Temporal`-style durable orchestration adjacency | Bain AU mentions workflow/orchestration + robust TS service contracts; Klue snippet names `Temporal`; Caribou remained in prior baseline | Durable execution is becoming easier to justify as a real stack dimension, not just a one-off. |
| `FastMCP` | Improbable snippet, Klue snippet | Distinct from generic `MCP`; suggests teams are now naming implementation libraries. |

### Stronger than last week

- `MCP`
  - Still a cross-region signal, but now appears with more implementation specificity: MCP servers, connector architecture, shipping libraries, FastMCP, AI-optimized APIs.
- Safety engineering language
  - `prompt injection`, `tool misuse`, `data exfiltration`, `red-teaming`, approval gates, and `human-in-the-loop` showed up more often than simple "guardrails" language.
- Bilingual runtime expectation
  - More roles now explicitly want `Python` plus `TypeScript`/`Node.js`, not one or the other.

### Weaker or still thin

- `Braintrust`
  - Still present, but remains materially less common than `LangSmith`/`Langfuse` in this sample.
- `Milvus`
  - Still effectively absent.
- `Weaviate`
  - Visible only as an option-list signal, not a strong primary dependency signal.
- `TensorZero`, `Helicone`
  - Still niche versus the `LangSmith`/`Langfuse`/`OpenTelemetry` cluster.

### Trend call

The direction that now looks more real than last week: the market is standardizing around agent runtime quality controls rather than agent framework ideology. Postings increasingly care about evals, observability, safe tool execution, approval paths, and typed/full-stack contracts more than they care about choosing one winning framework.

## Signal Map

### Category frequency in sampled postings

| Category | Stronger signals | Weaker / niche signals | Read |
| --- | --- | --- | --- |
| Agent frameworks | `LangGraph`, `LangChain`, `MCP` | `CrewAI`, `LlamaIndex`, `OpenAI Agents SDK`, `Google ADK`, `PydanticAI`, `Mastra`, `AutoGen`, `Claude Agent SDK` | `LangGraph` + `MCP` is the current center of gravity. |
| Eval / observability | `LangSmith`, `Langfuse`, `OpenTelemetry` | `Braintrust`, `Arize Phoenix`, `Promptfoo`, `Helicone`, custom eval frameworks | Vendor mix is widening, but `LangSmith`/`Langfuse` are still the safest default bet. |
| Retrieval / storage | `Pinecone`, `pgvector`, `Snowflake` | `Qdrant`, `Weaviate`, `FAISS`, `Neo4j`/`GraphRAG` | `Neo4j` appeared as a concrete graph signal this week. |
| Cloud / model platforms | `AWS Bedrock`, `Azure`, `OpenAI`, `Anthropic` | `Vertex AI`, `Databricks`, `Google ADK` as platform-adjacent | Enterprise roles still name cloud/model vendors directly. |
| Inference / model serving | `vLLM` | `SGLang`, `TGI`, `TensorRT-LLM`, `GGUF`, `ONNX`, `LoRA/PEFT`, quantization | Deeper inference-engineering signal remains US-heavy. |
| Safety / robustness | `guardrails`, `human-in-the-loop`, `prompt injection`, regression gates | `red-teaming`, threat modeling, approval routing, confidence thresholds | Safety vocabulary is becoming product-engineering vocabulary, not just safety-team vocabulary. |
| Languages / runtimes | `Python`, `TypeScript`, `Node.js`, `SQL` | `Go`, `Rust`, `C#`, `Scala`, `Java`, `C++` | Bilingual `Python` + `TypeScript` is stronger than TS-only or Python-only. |

## Company -> Tool Matrix

### Australia

| Company | Role | Concrete signal |
| --- | --- | --- |
| Bain (`DataEdge`) | Full Stack AI Product Engineer | `TypeScript`-first; `Node.js`; `Python`; `LangGraph`; `FastAPI`; `Docker`; `Kubernetes`; `Express`; `Bun`; `Koa`; `Fastify`; `NestJS`; `tRPC`; `Zod`; `OpenAPI`; `yarn`; `pnpm`; `ESLint`; `Prettier`; `Vitest`; `Jest`; `Vite`; `Turbopack`; `esbuild`; evaluation gates; regression controls; human-in-the-loop review. Source: [Bain](https://www.bain.com/careers/find-a-role/position/?jobid=106304) |
| Sonder (Sydney, posted 2026-05-12) | Fullstack AI Software Engineer | `Arize Phoenix`; `LangSmith`; `MCP`; `Pinecone`; `pgvector`; `CrewAI`; `LangChain`; `LangGraph`; `LlamaIndex`; `Next.js`; `Node.js`; `Python`; `SQL`; `Snowflake`; `dbt`. Source: [Sonder](https://agentic-engineering-jobs.com/jobs/sonder-fullstack-ai-software-engineer-agentic-systems-fullstack-builder-bvuu69) |
| Catapult Sports | Principal AI Engineer | `Go`; `Rust`; `C++`; `Python`; `TypeScript`; `Neo4j`; `GraphRAG`; community detection; `Promptfoo`; custom eval frameworks; automated scoring; regression detection; `AWS Bedrock`; multi-model orchestration. Source: [Catapult Sports](https://job-boards.greenhouse.io/catapultsports/jobs/7902420) |
| Karbon (Melbourne) | Associate AI Engineer | `RAG`; chaining; `MCP`; `Python`; `sklearn`; `PyTorch`; `TensorFlow`; `spaCy`; `Google ADK`; `LangGraph`; Agent SDK; `Snowflake`; `dbt`; `Azure`; `C#`. Source: [Karbon](https://job-boards.greenhouse.io/karbon/jobs/5995013004) |
| FutureSecureAI (Sydney) | Senior AI Engineer / Data Science | `LangChain`; `LlamaIndex`; `RAG`; multi-agent systems; tool-calling architectures; evaluation; benchmarking; safety. Source: [FutureSecureAI](https://job-boards.anz.greenhouse.io/futuresecureai/jobs/4001501201) |

### United Kingdom

| Company | Role | Concrete signal |
| --- | --- | --- |
| Lawhive | Senior Applied AI Engineer | `Langfuse` for monitoring generations, prompt management, and measurement. Source: [Lawhive](https://jobs.ashbyhq.com/lawhive/e2813e6f-88b6-43d7-b487-f2a7c7f3a909) |
| Distyl AI (London) | AI Engineer | `LangChain`; `LlamaIndex`; `Guardrails`; `MCP`; agent frameworks. Source: [Distyl AI](https://jobs.ashbyhq.com/Distyl/26cc59d5-af3c-4ae7-9ee9-f70733e68dd7) |
| StackOne (London) | AI Engineer | `MCP`; shipping libraries; `LangChain`; `Vercel AI SDK`; `TensorFlow`; `PyTorch`. Source: [StackOne](https://jobs.ashbyhq.com/stackone/52c111d8-594c-4d94-859f-c12be2ecd857/application) |
| Percepta (London, posted 2026-03-23) | Applied AI Engineer | `LangGraph`; `Mastra`; Anthropic alliance; `AWS`. Source: [Percepta](https://agentic-engineering-jobs.com/jobs/percepta-applied-ai-engineer-europe-Q5zdcl) |
| Smartsheet (UK remote, posted 2026-05-21) | Senior Forward Deployed AI Engineer | `MCP`; `LangChain`; `AWS`; `JavaScript`; `Python`; `TypeScript`. Source: [Smartsheet](https://agentic-engineering-jobs.com/jobs/smartsheet-senior-forward-deployed-ai-engineer-remote-eligible-in-the-uk-n956Hc) |
| Motorway (London) | Principal Generative AI Engineer | prompt injection eval design called out explicitly. Source: [Motorway](https://jobs.ashbyhq.com/motorway/ad77f636-832c-4520-b294-398f99639f6c) |

### United States

| Company | Role | Concrete signal |
| --- | --- | --- |
| BLEN | AI Engineer | `MCP` integrations; evals; structured outputs; function/tool calling; guardrails; `LangGraph`; `CrewAI`; `OpenAI Agents SDK`; Claude tool use. Source: [BLEN](https://jobs.lever.co/blencorp/4b2e3689-9720-4785-b0fe-d09bd5325f74) |
| Drata | Senior Platform AI Engineer | MCP server development; AI-optimized API design; human-in-the-loop workflows; confidence thresholds; safety guardrails; `vLLM` mentioned in platform-adjacent role family. Source: [Drata](https://jobs.ashbyhq.com/drata/f0ab62fb-c0a8-4bf2-bfd6-9d9d2e68fb91) and [Drata - Senior AI Engineer](https://jobs.ashbyhq.com/drata/938b5802-714a-4843-bf43-4b03b3c0bf12) |
| Pair Team | Staff AI Engineer | `LangGraph`; `Mastra`; `Claude Agents SDK`; `Langfuse`. Source: [Pair Team](https://job-boards.greenhouse.io/pairteam/jobs/8470873002) |
| Matter Intelligence | Applied AI Engineer - Internal | Anthropic SDK / Claude API; `MCP`; skills; orchestration; `LangSmith`; `Langfuse`; `Braintrust`; `DeepEval`; prompt injection defenses; secret management; access control. Source: [Matter Intelligence](https://jobs.ashbyhq.com/matter-intelligence/64f53375-69fd-4a2b-b3d7-df553c0b7b55) |
| Pivotal Health | Staff Engineer, AI Platform | `OpenTelemetry`; `Langfuse`; `LangSmith`; `Braintrust`; `Helicone`; `Datadog`; `Grafana`. Source: [Pivotal Health](https://jobs.ashbyhq.com/pivotal-health/5481c071-b85d-46da-993c-a971d66818e2) |
| Fathom | AI Engineer - Model Performance | `vLLM`; `SGLang`; `TensorRT-LLM`; speculative decoding. Source: [Fathom](https://jobs.ashbyhq.com/fathom.video/faff982d-4fc5-47d9-9af8-6efad4f2b636) |
| Newfront | Senior AI Engineer | `vLLM`; `TGI`; `TensorRT-LLM`; quantization; `LoRA/PEFT`; on-prem deployment. Source: [Newfront](https://jobs.ashbyhq.com/newfront/03cb6d44-29d1-4f8f-b3b5-a330b97ffcdd) |
| Level | Senior Applied AI Engineer | `OpenAI Agents SDK`; `PydanticAI`; `Mastra`; `LlamaIndex`; `CrewAI`; adversarial inputs. Source: [Level](https://jobs.ashbyhq.com/level/16021400-6ce7-47c8-87ef-58baf6652b75) |
| Titan AI | Applied AI Engineer | `PydanticAI`; `AutoGen`; vector DBs; MCP / connector architecture; multi-agent or planner-based architectures. Source: [Titan AI](https://jobs.ashbyhq.com/titan-ai/297cf9a9-289d-4cd5-a4a1-1e051f6f5d64) |
| Benchling | Agentic AI Engineer | `LangGraph`; `MCP`; memory/state management; evaluation; observability; threat modeling for prompt injection, tool misuse, data exfiltration. Source: [Benchling](https://jobs.ashbyhq.com/benchling/d5896e95-fed2-4cd4-b104-1ea4df92f7d7) |

## Pairings That Look Real

| Pairing | Where seen | Why it matters |
| --- | --- | --- |
| `LangGraph` + `MCP` | Sonder AU, Benchling US, Smartsheet UK, BLEN US | Orchestration plus tool-surface standard is becoming a default shape. |
| `LangSmith` + `Langfuse` | Matter Intelligence US, prior baseline examples, broader snippets | Teams are willing to multi-home eval/trace stacks. |
| `OpenTelemetry` + AI eval vendors | Pivotal Health US | AI observability is merging into standard application telemetry, not staying separate. |
| `Neo4j` + `GraphRAG` + `Promptfoo` + `Bedrock` | Catapult Sports AU | Strong sign that graph retrieval and eval gates are no longer just research talking points. |
| `TypeScript` + `Node.js` + `LangGraph` + typed contracts (`tRPC`/`Zod`) | Bain AU | AI product roles are getting more opinionated about full-stack correctness and type safety. |
| `PydanticAI` + `Mastra` + `OpenAI Agents SDK` | Level US | Newer framework mix is entering production-facing hiring language. |
| `vLLM` + `TensorRT-LLM` + quantization / `LoRA/PEFT` | Fathom US, Newfront US | Distinguishes inference/platform engineering from generic applied AI product work. |
| `MCP` + `Vercel AI SDK` | StackOne UK, Slash prior baseline | Product teams are pairing connector protocols with frontend-oriented AI SDKs. |

## Language / Runtime Signal

| Runtime | Strength in sample | Typical role shape |
| --- | --- | --- |
| `Python` | dominant | orchestration, evals, model integration, backend AI services |
| `TypeScript` / `Node.js` | strong second | AI product engineering, agent consoles, workflows, integrations |
| `SQL` | common | warehouse-backed copilots, analytics, RAG over business data |
| `Go` / `Rust` | niche but sharper this week | systems-heavy AI infra, performance, graph/data pipelines |
| `C#` | still secondary but real in AU enterprise | product integration, incumbent backend estates |
| `Java` / `Scala` | supporting signal | enterprise data/platform roles, forward-deployed roles |

### Runtime pattern

- The most credible hiring shape remains `Python` + `TypeScript`, not either one alone.
- `TypeScript` is increasingly not just frontend garnish; it owns product surfaces, backend services, typed contracts, and orchestration-adjacent logic.
- `Go`/`Rust` showed up as serious secondary signals in Australia, which is stronger than last week's mostly Python/TS framing.

## Safety / Robustness Signal

### Common

- `guardrails`
- `human-in-the-loop`
- confidence thresholds
- regression gates
- structured outputs
- tool-use controls

### More specific this week

- prompt injection defenses
- threat modeling for tool misuse and data exfiltration
- adversarial inputs
- red-teaming
- approval routing / fallback paths

Examples:

- Benchling: prompt injection, tool misuse, data exfiltration. Source: [Benchling](https://jobs.ashbyhq.com/benchling/d5896e95-fed2-4cd4-b104-1ea4df92f7d7)
- Matter Intelligence: prompt injection defenses from day one. Source: [Matter Intelligence](https://jobs.ashbyhq.com/matter-intelligence/64f53375-69fd-4a2b-b3d7-df553c0b7b55)
- Motorway: design evals that catch prompt injection. Source: [Motorway](https://jobs.ashbyhq.com/motorway/ad77f636-832c-4520-b294-398f99639f6c)
- Level: enrich systems against adversarial inputs. Source: [Level](https://jobs.ashbyhq.com/level/16021400-6ce7-47c8-87ef-58baf6652b75)

## Seen / Not Seen

### Seen clearly

- `LangGraph`
- `MCP`
- `LangSmith`
- `Langfuse`
- `Pinecone`
- `pgvector`
- `Snowflake`
- `AWS Bedrock`
- `OpenAI Agents SDK`
- `Google ADK`
- `PydanticAI`
- `vLLM`
- `Promptfoo`
- `Neo4j`

### Seen, but not dominant

- `Mastra`
- `Arize Phoenix`
- `Braintrust`
- `Qdrant`
- `Weaviate`
- `LlamaIndex`
- `CrewAI`
- `Claude Agent SDK`

### Weak / absent in this sample

- `Milvus`
- `turbopuffer`
- `TensorZero`
- `DeepEval` outside a small number of US roles
- `Azure AI Search` as an explicit first-choice retrieval layer
- `Vertex AI` as a dominant applied-AI hiring dependency

## Regional Read

| Region | Strongest pattern | Distinctive signal this week |
| --- | --- | --- |
| Australia | full-stack applied AI with enterprise/product constraints | sharper graph+eval signal (`Neo4j`, `GraphRAG`, `Promptfoo`, `Bedrock`) and stronger TS-first product engineering (`tRPC`, `Zod`, `Vitest`, `Turbopack`) |
| United Kingdom | product/integration AI with monitoring and connector emphasis | `Langfuse`, `MCP`, `Vercel AI SDK`, `Mastra`, prompt-injection-aware roles |
| United States | widest spread from product AI to inference/platform AI | deepest eval/observability and inference-stack specificity (`OpenTelemetry`, `Helicone`, `vLLM`, `TensorRT-LLM`, quantization, `LoRA/PEFT`) |

## Source Set

- [Bain - Full Stack AI Product Engineer](https://www.bain.com/careers/find-a-role/position/?jobid=106304)
- [Sonder - Fullstack AI Software Engineer](https://agentic-engineering-jobs.com/jobs/sonder-fullstack-ai-software-engineer-agentic-systems-fullstack-builder-bvuu69)
- [Catapult Sports - Principal AI Engineer](https://job-boards.greenhouse.io/catapultsports/jobs/7902420)
- [Karbon - Associate AI Engineer](https://job-boards.greenhouse.io/karbon/jobs/5995013004)
- [FutureSecureAI - Senior AI Engineer, Data Science](https://job-boards.anz.greenhouse.io/futuresecureai/jobs/4001501201)
- [Lawhive - Senior Applied AI Engineer](https://jobs.ashbyhq.com/lawhive/e2813e6f-88b6-43d7-b487-f2a7c7f3a909)
- [Distyl AI - AI Engineer](https://jobs.ashbyhq.com/Distyl/26cc59d5-af3c-4ae7-9ee9-f70733e68dd7)
- [StackOne - AI Engineer](https://jobs.ashbyhq.com/stackone/52c111d8-594c-4d94-859f-c12be2ecd857/application)
- [Percepta - Applied AI Engineer](https://agentic-engineering-jobs.com/jobs/percepta-applied-ai-engineer-europe-Q5zdcl)
- [Smartsheet - Senior Forward Deployed AI Engineer](https://agentic-engineering-jobs.com/jobs/smartsheet-senior-forward-deployed-ai-engineer-remote-eligible-in-the-uk-n956Hc)
- [BLEN - AI Engineer](https://jobs.lever.co/blencorp/4b2e3689-9720-4785-b0fe-d09bd5325f74)
- [Matter Intelligence - Applied AI Engineer](https://jobs.ashbyhq.com/matter-intelligence/64f53375-69fd-4a2b-b3d7-df553c0b7b55)
- [Pivotal Health - Staff Engineer, AI Platform](https://jobs.ashbyhq.com/pivotal-health/5481c071-b85d-46da-993c-a971d66818e2)
- [Fathom - AI Engineer, Model Performance](https://jobs.ashbyhq.com/fathom.video/faff982d-4fc5-47d9-9af8-6efad4f2b636)
- [Newfront - Senior AI Engineer](https://jobs.ashbyhq.com/newfront/03cb6d44-29d1-4f8f-b3b5-a330b97ffcdd)
- [Level - Senior Applied AI Engineer](https://jobs.ashbyhq.com/level/16021400-6ce7-47c8-87ef-58baf6652b75)
- [Titan AI - Applied AI Engineer](https://jobs.ashbyhq.com/titan-ai/297cf9a9-289d-4cd5-a4a1-1e051f6f5d64)
- [Benchling - Agentic AI Engineer](https://jobs.ashbyhq.com/benchling/d5896e95-fed2-4cd4-b104-1ea4df92f7d7)
- [Motorway - Principal Generative AI Engineer](https://jobs.ashbyhq.com/motorway/ad77f636-832c-4520-b294-398f99639f6c)
