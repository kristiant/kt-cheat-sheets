# State of AI Engineering

Date: 2026-06-22
Scope: live sample of AI Engineer / Applied AI Engineer / AI Product Engineer / ML Engineer postings across Australia, the US, and the UK reviewed on 2026-06-22.
Method: treat postings as telemetry; record concrete named stack signals and recurring implementation patterns.

## Week-over-Week Change

### New or clearer this week

| Signal | Evidence | Read |
| --- | --- | --- |
| `DSPy` | Temple & Webster AU | First clear AU product-role appearance in this automation. |
| `Langflow` | Plenti AU | Distinct from the dominant `LangGraph` track; paired with infra ownership. |
| Dataiku agent platform surface | Dataiku UK: `Agent Hub`, `Prompt Studio`, `LLM Mesh`, `Knowledge Banks`, `Cost Guard`, `Quality Guard`, `A2A` | Strong new enterprise platform signal, not just framework signal. |
| `Strands Agents` | Forter UK | New named orchestration option in a production-facing role. |
| `Deep Agents` | Awin UK | New named agent-runtime family in an explicit production role. |
| AI coding tools as job requirements | Forter UK, Dataiku UK, Machinify US: `Claude Code`, `Cursor`, `GitHub Copilot`, `Codex` | Shift from "nice to have" to named hiring signal. |
| `Semantic Kernel` + `Ollama` + `Ray` | Accenture Federal US | Clearer Microsoft/open-model enterprise mix than prior sample. |
| Voice-agent stack specifics | Zoom US: `ElevenLabs`, `Azure`, `Cartesia`, `Genesys`, `Five9`, `NICE`, `Decagon`, `Sierra` | New concrete voice-agent vendor cluster. |
| Post-training / alignment stack | OKX US: `DPO`, `GRPO`, `RLAIF`, reward models, `vLLM`, `SGLang` | Stronger than prior generic inference tuning language. |
| `Inngest` + `Temporal` | Machinify US | Durable orchestration is now directly named again, not inferred. |
| `turbopuffer` | Machinify US | Moved from absent last week to named vector-store option. |

### Stronger than last week

- `MCP`
  - Still broad, but more often framed as an internal server surface exposing enterprise systems rather than generic protocol literacy.
- Eval-first engineering
  - More roles explicitly require eval harnesses, scripted/adversarial scenarios, reproducibility, regression testing, or scoring rubrics.
- AI-assisted engineering workflow
  - `Claude Code`, `Codex`, `Cursor`, and `GitHub Copilot` now appear as part of the working model of an AI engineer.
- Inference and deployment specialization
  - More current roles name `vLLM`, `SGLang`, `TensorRT-LLM`, quantization, model routing, or GPU-serving tradeoffs directly.

### Weaker or newly absent versus last week

- `Promptfoo`
  - Strong AU appearance last run; not resurfacing in this week's refreshed sample.
- `Neo4j`
  - Last week had a very sharp `Neo4j` + `GraphRAG` AU signal; this week `GraphRAG` remains, but `Neo4j` did not recur in the sampled set.
- `Braintrust`
  - Still visible in the broader market, but not a strong signal in this week's sampled postings.
- `Google ADK`
  - Still present in AU through Karbon, but less recurrent than last week.

### Trend call

What looks more real now than on 2026-06-15: production AI engineering roles are converging on three concrete shapes:

1. product-agent engineers with `Python` + `TypeScript`, evals, tracing, and tool-use safety
2. enterprise internal-AI engineers building `MCP` server surfaces over existing systems
3. inference/platform engineers naming serving stacks, GPU deployment tradeoffs, and post-training methods directly

## Signal Map

### Category strength in this sample

| Category | Stronger signals | Emerging / niche signals | Read |
| --- | --- | --- | --- |
| Agent frameworks / runtimes | `LangGraph`, `MCP`, `OpenAI Agents SDK`, `Claude Agent SDK` | `DSPy`, `Langflow`, `CrewAI`, `Deep Agents`, `Strands Agents`, `Semantic Kernel` | `LangGraph` stays central, but the edge of the market is widening. |
| Eval / observability | `Langfuse`, `OpenTelemetry`, custom eval harnesses, regression tests | `LangSmith`, LLM-as-a-judge, rubric scoring, adversarial scenarios | Evals are now part of job mechanics, not just platform tooling. |
| Retrieval / storage | `pgvector`, `Pinecone`, vector DBs broadly | `Weaviate`, `turbopuffer`, hybrid retrieval, GraphRAG, memory systems | More teams are naming retrieval choices or at least demanding that judgment. |
| Cloud / model platforms | `AWS Bedrock`, `Azure`, `OpenAI`, `Anthropic`, `GCP/Vertex` | `Azure AI Foundry`, open-source models via vendor meshes | Multi-provider routing is more explicit this week. |
| Inference / serving | `vLLM`, `SGLang` | `TensorRT-LLM`, `Ollama`, `Ray`, quantization, `TGI` | US roles continue to dominate this depth. |
| Safety / robustness | guardrails, eval gates, human approval, prompt-injection defenses | red-teaming, context leakage, content safety, policy enforcement, confidence thresholds | Safety vocabulary is now engineering-operational language. |
| Languages / runtimes | `Python`, `TypeScript`, `Node.js` | `C#/.NET`, `Go`, `JavaScript`, `Go`, `Ray`-adjacent infra | `Python` + `TypeScript` remains the most credible baseline. |

## Company -> Tool Matrix

### Australia

| Company | Role | Concrete signal |
| --- | --- | --- |
| Temple & Webster | AI Implementation Engineer | `LangGraph`; `DSPy`; Google + Anthropic models; `Python`; `GCP`; `LLMOps`; autonomous agents. Source: [SEEK](https://au.seek.com/ai-engineer-jobs/in-Darlinghurst-NSW-2010) |
| Plenti | AI Engineer | `Python`; `Langflow`; agentic workflows; specialized `Kubernetes` nodes; model orchestration environments; testing/monitoring focus. Source: [Plenti](https://jobs.lever.co/plenti/2a0ab049-b8d3-4db5-bce9-acfcce9173e2) |
| Karbon | Associate AI Engineer | `MCP`; `RAG`; chaining; `ADK`; `LangGraph`; Agent SDK; `Snowflake`; `DBT`; `Azure`; `Datadog`; `C#/.NET`; `TypeScript`; Azure Container Apps. Source: [Karbon](https://job-boards.greenhouse.io/karbon/jobs/5995013004) |
| Top-tier bank via Robert Walters | AI/Data Scientist - Production grade LLM & Agentic systems | Anthropic + `AWS` tools; AI COE; production LLM and agentic systems. Source: [SEEK](https://www.seek.com.au/artificial-intelligence-jobs) |

### United Kingdom

| Company | Role | Concrete signal |
| --- | --- | --- |
| Dataiku | Senior Generative AI Engineer | `Agent Hub`; `Prompt Studio`; `LLM Mesh`; `Knowledge Banks`; `LangGraph`; `CrewAI`; `Claude Agent SDK`; `OpenAI Agents SDK`; `MCP`; GraphRAG; reranking; model routing; `Cost Guard`; `Quality Guard`; `Vue.js`; `Node.js`; `A2A`; `Claude Code`; `Cursor`; `GitHub Copilot`. Source: [Dataiku](https://job-boards.greenhouse.io/dataiku/jobs/5978968004) |
| Forter | Senior Software AI Engineer | internal `MCP` server; AI guardrails; evaluation frameworks; `RAG`; `Claude Code`; `GitHub Copilot`; `Cursor`; skills; `AWS` / `GCP` / `Azure`. Source: [Forter](https://job-boards.greenhouse.io/forter/jobs/8415897002) |
| Forter | Senior Applied AI Engineer | shared `MCP` servers; `RAG`; evaluation frameworks; `Claude Code`; `Strands Agents`; AI enablement for non-R&D teams. Source: [Forter](https://job-boards.greenhouse.io/forter/jobs/8486828002) |
| Awin | Senior AI Engineer | `LangGraph`; `Deep Agents`; `LangSmith`; `MCP`; vector DBs; hybrid retrieval; persistent memory; human-in-the-loop approval; prompt-injection and context-leakage mitigation; `AWS ECS`; `Lambda`; `S3`; `API Gateway`; `PostgreSQL`; `Redis`; `Docker`; `GitHub Actions`. Source: [Awin](https://job-boards.greenhouse.io/awin/jobs/7720582003) |

### United States

| Company | Role | Concrete signal |
| --- | --- | --- |
| Future | Applied AI Engineer | `Langfuse`; `LangGraph`; `LangChain`; `Pydantic`; `JSON Schema`; `SSE`; webhooks; prompt caching; token budgets; `AWS Bedrock`; `OpenTelemetry`; `Datadog`; `Terraform`; `CDK`; LLM-as-a-judge. Source: [Future](https://job-boards.greenhouse.io/future/jobs/4683133005) |
| Zoom | Applied AI Engineers | eval frameworks with scripted + adversarial scenarios; tool use; multi-step orchestration; guardrails; `ElevenLabs`; `Azure`; `Cartesia`; `Python`; `TypeScript`; `Go`; `Genesys`; `Five9`; `NICE`; `Zoom Contact Center`; `Decagon`; `Sierra`. Source: [Zoom](https://careers.zoom.us/jobs/applied-ai-engineers-remote-united-states-san-jose-california-seattle-washington) |
| Fireworks AI | AI Field Engineer - Enterprise | `vLLM`; `SGLang`; `TensorRT-LLM`; quantization configs; `SFT`; `DPO`; `RFT`; `Azure AI Foundry`; `AWS Bedrock`; `SageMaker`; `Vertex`; GPU infra; `Kubernetes`. Source: [Fireworks AI](https://job-boards.greenhouse.io/fireworksai/jobs/4284317009) |
| Machinify | Staff AI Engineer | `Python`; `TypeScript`; orchestration via custom / `LangGraph` / `Inngest` / `Temporal`; vector stores `pgvector` / `Pinecone` / `turbopuffer` / `Weaviate`; `MCP`; `vLLM`; `TGI`; `SGLang`; red-teaming. Source: [Machinify](https://job-boards.greenhouse.io/machinifyinc/jobs/4212332009) |
| Machinify | Staff AI Engineer \| Agentic Systems | `OpenAI Agents SDK`; Anthropic SDK / `claude-agent-sdk`; `LangGraph`; eval-first shipping; citation grounding; `Claude Code`; `Codex`; reasoning models (`o-series`, Claude extended thinking, Gemini thinking); cost-aware model selection. Source: [Machinify](https://job-boards.greenhouse.io/machinifyinc/jobs/4146863009) |
| Accenture Federal Services | AI Engineer | `LangGraph`; `Semantic Kernel`; `vLLM`; `Ollama`; `Ray`; NVIDIA GPU ecosystems; vector DBs; `Kubernetes`; RAG; observability; cybersecurity; hub-and-spoke architecture. Source: [Accenture Federal Services](https://job-boards.greenhouse.io/accenturefederalservices/jobs/4683202006?gh_jid=4683202006) |
| Govini | AI Engineer | `Claude Agent SDK`; `OpenAI Agent SDK`; multi-agent coordination; routing; tool orchestration; Agent Skills; observability; evaluation; regression testing; reliability metrics. Source: [Govini](https://job-boards.greenhouse.io/govini/jobs/4280882009) |
| OKX | Staff/Senior Staff AI Engineer, Model Post-Training and Alignment | `DPO`; `GRPO`; `RLAIF`; reward models; post-training pipelines; `vLLM`; `SGLang`; low-latency deployment. Source: [OKX](https://job-boards.greenhouse.io/okx/jobs/7652671003) |
| Matter Intelligence | Applied AI Engineer - Internal | Anthropic SDK / Claude API; `MCP`; skills; `n8n`; `Zapier`; `Make`; `Retool`; orchestration tools. Source: [Matter Intelligence](https://jobs.ashbyhq.com/matter-intelligence/64f53375-69fd-4a2b-b3d7-df553c0b7b55) |

## Pairings That Look Real

| Pairing | Where seen | Why it matters |
| --- | --- | --- |
| `LangGraph` + `DSPy` + Google/Anthropic | Temple & Webster AU | Shows experimentation with newer reasoning/prompting stacks in practical product roles. |
| `Langflow` + `Kubernetes` + model orchestration | Plenti AU | Low-code orchestration is showing up alongside real infra ownership, not just prototypes. |
| `MCP` + internal enterprise server surfaces | Forter UK, Dataiku UK, Karbon AU | `MCP` is increasingly an integration layer over internal tools/data, not just protocol familiarity. |
| `Langfuse` + `OpenTelemetry` + `Datadog` | Future US | AI tracing is merging with standard app telemetry. |
| `LangGraph`/agent SDKs + eval-first shipping | Future US, Govini US, Machinify US, Dataiku UK | Production AI teams increasingly define the stack around eval discipline, not framework choice. |
| `vLLM` + `SGLang` + `TensorRT-LLM` + hyperscaler AI platforms | Fireworks AI US | Forward-deployed inference tuning is now explicit hiring language. |
| `Semantic Kernel` + `Ollama` + `Ray` + NVIDIA GPU ecosystem | Accenture Federal US | New concrete blend of enterprise orchestration, open-model deployment, and GPU infra. |
| `Claude Code` + `Cursor` + `Copilot` + agent frameworks | Forter UK, Dataiku UK, Machinify US | AI-assisted development workflow is entering the stack definition itself. |
| `Temporal`/`Inngest` + vector-store choice | Machinify US | Durable workflow tooling is pairing with agent orchestration and retrieval decisions. |
| `DPO` + `GRPO` + `RLAIF` + `vLLM`/`SGLang` | OKX US | Strong sign that some "AI Engineer" roles are now materially post-training/alignment roles. |

## Language / Runtime Signal

| Runtime | Strength in sample | Notes |
| --- | --- | --- |
| `Python` | dominant | still the default control plane for agents, evals, integrations, and serving workflows |
| `TypeScript` / `Node.js` | strong | stronger again this week, especially in product, internal tooling, and full-stack delivery |
| `JavaScript` frontend stack | present | explicit in Dataiku UK (`Vue.js`, `Node.js`) and Karbon AU (`TypeScript`, React, React Native) |
| `C#/.NET` | real in AU | Karbon makes the incumbent enterprise runtime explicit rather than treating AI as a separate stack |
| `Go` | supporting | appears in Zoom requirements alongside Python/TS, suggesting systems and voice infra adjacency |
| workflow runtimes | stronger | `Temporal`, `Inngest`, streaming via `SSE`, webhooks, agent routing/planning |

### Runtime pattern

- The baseline role is still `Python` plus enough `TypeScript`/`Node.js` to ship productized systems.
- Pure prompt-engineering language is weak; postings expect backend/service competence, observability, and deployment literacy.
- Enterprise environments are not hiding incumbent stacks anymore: `.NET`, `Azure`, `Datadog`, `GitHub Actions`, `Redis`, `PostgreSQL`, `Kubernetes`, and `Terraform` are named directly.

## Safety / Robustness Signal

### Common

- guardrails
- evaluation frameworks
- regression testing
- structured outputs
- human approval / HITL
- cost and latency control

### More specific this week

- adversarial scenarios as deployment gates
- prompt-injection defenses
- context leakage mitigation
- red-teaming
- policy enforcement
- confidence / reliability scoring
- citation grounding

Examples:

- Zoom: scripted tasks plus adversarial scenarios used as deployment gates. Source: [Zoom](https://careers.zoom.us/jobs/applied-ai-engineers-remote-united-states-san-jose-california-seattle-washington)
- Awin: prompt injection and context leakage called out explicitly. Source: [Awin](https://job-boards.greenhouse.io/awin/jobs/7720582003)
- Machinify: red-teaming appears as a relevant preferred capability. Source: [Machinify](https://job-boards.greenhouse.io/machinifyinc/jobs/4212332009)
- Dataiku: reliability/accuracy/cost evals plus governance and responsible-AI constraints. Source: [Dataiku](https://job-boards.greenhouse.io/dataiku/jobs/5978968004)

## Seen / Not Seen

### Seen clearly

- `LangGraph`
- `MCP`
- `Langfuse`
- `OpenAI Agents SDK`
- `Claude Agent SDK`
- `AWS Bedrock`
- `vLLM`
- `SGLang`
- `Python`
- `TypeScript`
- eval harnesses / regression tests / observability

### Seen, but still not baseline

- `DSPy`
- `Langflow`
- `Semantic Kernel`
- `Ollama`
- `Strands Agents`
- `Deep Agents`
- `turbopuffer`
- `Weaviate`
- `TensorRT-LLM`
- `A2A`

### Weak / absent in this sample

- `Promptfoo`
- `Braintrust`
- `Neo4j`
- `Milvus`
- `TensorZero`
- `Helicone`
- `Qdrant`

## Regional Read

| Region | Strongest pattern | Distinctive signal this week |
| --- | --- | --- |
| Australia | pragmatic product/internal AI built inside incumbent enterprise stacks | `DSPy`, `Langflow`, `GCP`, Anthropic, `Kubernetes`, `.NET`, `Azure`, `Datadog`, `Snowflake`, `DBT` |
| United Kingdom | enterprise internal-AI and enablement roles with heavier platform vocabulary | Dataiku platform surface, `MCP` servers, `Cost Guard`, `Quality Guard`, `Strands Agents`, `Deep Agents`, explicit AI coding-tool fluency |
| United States | widest spread from product agents to field inference to post-training | voice-agent vendors, `Semantic Kernel`/`Ollama`, `Temporal`/`Inngest`, GPU-serving stacks, `DPO`/`GRPO`/`RLAIF` |

## Direction

- `MCP` is no longer just "knows the protocol"; it is increasingly "build and own the server surface over enterprise tools and data."
- The center of gravity is still agent/application engineering, but more roles now separate product-agent work from inference/platform work.
- AI engineering is becoming more operationally explicit: tracing, eval reproducibility, token budgets, routing, caching, and workflow durability are now named expectations.
- AI-assisted development tools are entering the role definition itself, which is a stronger signal than last week.

## Source Set

- [Temple & Webster via SEEK](https://au.seek.com/ai-engineer-jobs/in-Darlinghurst-NSW-2010)
- [Plenti](https://jobs.lever.co/plenti/2a0ab049-b8d3-4db5-bce9-acfcce9173e2)
- [Karbon](https://job-boards.greenhouse.io/karbon/jobs/5995013004)
- [SEEK - Artificial Intelligence Jobs](https://www.seek.com.au/artificial-intelligence-jobs)
- [Dataiku](https://job-boards.greenhouse.io/dataiku/jobs/5978968004)
- [Forter - Senior Software AI Engineer](https://job-boards.greenhouse.io/forter/jobs/8415897002)
- [Forter - Senior Applied AI Engineer](https://job-boards.greenhouse.io/forter/jobs/8486828002)
- [Awin](https://job-boards.greenhouse.io/awin/jobs/7720582003)
- [Future](https://job-boards.greenhouse.io/future/jobs/4683133005)
- [Zoom](https://careers.zoom.us/jobs/applied-ai-engineers-remote-united-states-san-jose-california-seattle-washington)
- [Fireworks AI](https://job-boards.greenhouse.io/fireworksai/jobs/4284317009)
- [Machinify - Staff AI Engineer](https://job-boards.greenhouse.io/machinifyinc/jobs/4212332009)
- [Machinify - Staff AI Engineer | Agentic Systems](https://job-boards.greenhouse.io/machinifyinc/jobs/4146863009)
- [Accenture Federal Services](https://job-boards.greenhouse.io/accenturefederalservices/jobs/4683202006?gh_jid=4683202006)
- [Govini](https://job-boards.greenhouse.io/govini/jobs/4280882009)
- [OKX](https://job-boards.greenhouse.io/okx/jobs/7652671003)
- [Matter Intelligence](https://jobs.ashbyhq.com/matter-intelligence/64f53375-69fd-4a2b-b3d7-df553c0b7b55)
