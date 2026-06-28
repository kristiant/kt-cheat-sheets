# State of AI Engineering

Date: 2026-06-29
Compared against prior run: 2026-06-23
Sample: 16 current postings across Australia, the US, and the UK
Method: treat postings as telemetry; prefer official job pages; record concrete named stack details only

## Week-over-Week Change

### New or materially clearer this week

| Signal | Evidence | Read |
| --- | --- | --- |
| `LiteLLM` | Omada US; Smartsheet US | Multi-model routing and gateway layering are becoming explicit job requirements instead of hidden implementation detail. |
| AI gateway layer | Omada US (`Kong AI Gateway`); Smartsheet US (`Kong AI Gateway`) | Gateway products are showing up alongside observability, PII controls, and spend controls. |
| named eval vendors | Smartsheet US (`DeepEval`, `RAGAS`); Awin UK (`LangSmith`) | The market is getting less generic about evals. |
| agent SDK standardization | Dataiku UK (`OpenAI Agents SDK`, `Claude Agent SDK`); PlayStation US (`OpenAI Agents SDK`, `Semantic Kernel`) | More roles now ask for a specific agent runtime, not just "agentic AI". |
| `A2A` alongside `MCP` | Dataiku UK | Protocol language is widening beyond MCP-only stacks. |
| inference serving stack depth | Fireworks US (`vLLM`, `SGLang`, `TensorRT-LLM`); Assail US (`quantization`, `GRPO`) | US-side infra roles are much more explicit about serving and post-training internals this week. |
| `Vercel AI SDK` | DHI US | This is the clearest fresh web-product signal in the sample. |
| `n8n` in production AI roles | PlayStation US | Automation tooling is being named directly inside enterprise agent workflows. |
| `Azure Container Apps` + `Azure AI Search` | Karbon AU | AU postings are staying cloud-platform specific, not only generic RAG wording. |
| `Databricks Vector Search` | Smartsheet US | Databricks is showing up not just as a lakehouse but as a retrieval substrate. |

### Stronger than last week

- `MCP` remains strong, but now appears more often next to policy layers: auth, RBAC, gateways, catalog/governance, and enterprise connectors.
- eval/observability stacks are more named and more operational: `LangSmith`, `DeepEval`, `RAGAS`, `Datadog`, `OpenTelemetry`.
- `TypeScript`/`Node.js` remains a first-class AI runtime, especially in product-engineering, internal tools, and protocol/server layers.
- AU signal still splits cleanly between agent-product engineering and deep model/training infra.

### Weaker, thinner, or not reinforced this week

- `Promptfoo`
- `Braintrust`
- `Langfuse`
- `Phoenix`
- `TensorZero`
- `Helicone`
- `CrewAI` appears, but only once
- `AutoGen` is still present, but not the center of gravity in fresh postings

### Direction call

The clearest direction now: AI engineering jobs are standardizing around three layers at once.

1. Protocol and tool-access layer: `MCP`, `A2A`, agent SDKs, internal servers
2. Control plane layer: evals, tracing, gateways, policy, cost, PII, auth
3. Data/retrieval layer: vector search, graph retrieval, governed enterprise systems

This is more concrete than last week. The new signal is not just "agents"; it is controlled agent infrastructure.

## Signal Map

### Common in this sample

| Category | Frequent signals | Read |
| --- | --- | --- |
| Languages / runtimes | `Python`, `TypeScript`/`Node.js`, `SQL` | `Python` remains dominant; `TypeScript` is firmly established as a production AI runtime. |
| Orchestration / protocols | `MCP`, `LangGraph`, function/tool calling, multi-agent workflows | `MCP` is still the most visible protocol signal across regions. |
| Model platforms | `OpenAI`, `Anthropic`, `AWS Bedrock`, `Azure OpenAI` | Closed-model APIs remain the default application layer. |
| Retrieval / storage | `RAG`, vector databases, hybrid retrieval, `pgvector`, `Azure AI Search`, graph retrieval | Retrieval remains standard; the named substrate varies by company maturity. |
| Eval / observability | tracing, regression suites, `LangSmith`, `DeepEval`, `RAGAS`, `Datadog`, `OpenTelemetry` | Evals are moving closer to normal software delivery discipline. |
| Safety / robustness | RBAC, `OAuth2`, guardrails, prompt-injection defenses, jailbreak resistance, audit logging | Governance language keeps getting more explicit. |

### Distinct but still niche

| Category | Signals |
| --- | --- |
| Inference serving | `vLLM`, `SGLang`, `TensorRT-LLM` |
| Post-training / model adaptation | `LoRA`, `QLoRA`, `GRPO`, `SFT`, `RLHF` |
| Graph-heavy reasoning | `GraphRAG`, `Neo4j`, graph traversal |
| Enterprise connector mesh | `Salesforce`, `NetSuite`, `ServiceNow`, `SAP`, `Oracle` |
| AI gateway layer | `Kong AI Gateway`, `LiteLLM` |
| Training infra | `TorchTitan`, `Megatron-LM`, `FSDP`, `DeepSpeed`, `NCCL`, `Slurm` |

## Company -> Tool Matrix

### Australia

| Company | Role | Concrete signal |
| --- | --- | --- |
| Bain (`DataEdge`) | Full Stack AI Product Engineer | `TypeScript`; `Node.js`; `Python`; `LangGraph`; `FastAPI`; `Pydantic v2`; `Temporal`; `pgvector`; `OpenSearch`; `Docker`; `Kubernetes`; `tRPC`; `Zod`; `OpenAPI codegen`. Source: [Bain](https://www.bain.com/careers/find-a-role/position/?jobid=106304) |
| Karbon | Senior AI Engineer | `Azure`; `Azure AI Search`; `Azure Container Apps`; `MCP`; `LangGraph`; `RAG`; `Python`; `TypeScript`; `C#`; `Snowflake`; `dbt`. Source: [Karbon](https://job-boards.greenhouse.io/karbon/jobs/6040484004) |
| StarRez | AI Engineer | `TypeScript`; `Node.js`; `Python`; `AWS`; `OpenAI`; `Anthropic`; `RAG`; vector databases; observability; prompt engineering. Source: [StarRez](https://job-boards.greenhouse.io/starrez/jobs/8165138002) |
| Caterpillar | Agentic AI / AI Ops Engineer | `Kubernetes`; telemetry pipelines; logs/metrics/traces; `LangGraph`; `AutoGen`; multi-step workflows; tool integration; reliability framing. Source: [Caterpillar](https://careers.caterpillar.com/en/jobs/r0000376964/agentic-ai-ai-ops-engineer-platform-engineering/) |
| Catapult Sports | Principal AI Engineer | `Go`; `Rust`; `Python`; `Neo4j`; `GraphRAG`; `AWS Bedrock`; `LLM-as-judge`; custom eval frameworks. Source: [Catapult Sports](https://job-boards.greenhouse.io/catapultsports/jobs/7902420) |
| Firmus Technologies | AI Engineer, AI & Applications | `TorchTitan`; `Megatron-LM`; `FSDP`; `NCCL`; `PyTorch`; `JAX`; `DeepSpeed`; `Kubernetes`; `Slurm`; `LoRA`; `QLoRA`; offline evaluation harnesses. Source: [Firmus](https://job-boards.greenhouse.io/firmus/jobs/5071736008) |

### United Kingdom

| Company | Role | Concrete signal |
| --- | --- | --- |
| Awin | Senior AI Engineer | `AWS`; `LangGraph`; deep agents; `LangSmith`; `MCP`; vector databases; hybrid retrieval; structured evals; tracing/observability; caching and routing. Source: [Awin](https://job-boards.greenhouse.io/awin/jobs/7720582003) |
| Forter | Senior Applied AI Engineer | `MCP`; `RAG`; evaluation frameworks; guardrails; `Claude Code`; `Cursor`; `GitHub Copilot`; cloud on `AWS` / `GCP` / `Azure`; business-embedded deployment. Source: [Forter](https://job-boards.greenhouse.io/forter/jobs/8486828002) |
| Dataiku | Applied AI Engineer | `OpenAI Agents SDK`; `Claude Agent SDK`; `MCP`; `A2A`; `CrewAI`; `LangGraph`; `GraphRAG`; `Pinecone`; `Neo4j`; `Milvus`; `Redis`; `Azure AI Search`; `Databricks`; `AWS Bedrock`; `Azure OpenAI`; `Gemini`. Source: [Dataiku](https://job-boards.greenhouse.io/dataiku/jobs/5978968004) |
| IFS | Forward Deployed AI Engineer | `OpenAI`; `Hugging Face`; `LangChain`; `Pinecone`; `Weaviate`; `Salesforce`; `ServiceNow`; `SAP`; `Oracle`; multi-agent orchestration in enterprise systems. Source: [IFS](https://jobs.smartrecruiters.com/IFS1/744000128816717-forward-deployed-ai-engineer) |

### United States

| Company | Role | Concrete signal |
| --- | --- | --- |
| Smartsheet | Sr. Forward Deployed AI Engineer | `MCP`; `LiteLLM`; `Kong AI Gateway`; `DeepEval`; `RAGAS`; `AWS Bedrock`; `Databricks`; `Databricks Vector Search`; `Temporal`; `Python`; `TypeScript`; PII masking; cost tracking; evaluation gates. Source: [Smartsheet](https://job-boards.greenhouse.io/smartsheet/jobs/7930145) |
| PlayStation | Applied AI Engineer | `OpenAI Agents SDK`; `LangGraph`; `Semantic Kernel`; `AWS`; `Python`; `Node.js`; `n8n`; `MCP`; browser automation; observability; enterprise workflow agents. Source: [PlayStation](https://boards.greenhouse.io/sonyinteractiveentertainmentglobal/jobs/5616918004) |
| Fireworks AI | AI Infrastructure Engineer | `Python`; `C++`; `CUDA`; `vLLM`; `SGLang`; `TensorRT-LLM`; distributed inference; GPU serving optimization. Source: [Fireworks AI](https://job-boards.greenhouse.io/fireworksai/jobs/4280748009) |
| Omada Health | Staff Software Engineer, AI Platform | `MCP`; `LiteLLM`; `Kong AI Gateway`; `AWS`; `Datadog`; `Karpenter`; `Kubernetes`; `Terraform`; `Python`; `TypeScript`; policy/compliance controls. Source: [Omada Health](https://job-boards.greenhouse.io/omadahealth/jobs/4738058008) |
| Assail | AI Engineer | `Python`; `Kubernetes`; `SFT`; `RLHF`; `GRPO`; quantization; co-evolutionary self-training; model adaptation and optimization. Source: [Assail](https://jobs.ashbyhq.com/assail/7d92138f-7785-49df-a1d8-e757e84a4ebc) |
| DHI Group | Lead AI Engineer | `TypeScript`; `Node.js`; `Python`; `Vercel AI SDK`; `MCP`; `LangGraph`; `OpenAI`; `Anthropic`; `Azure OpenAI`; `Qdrant`; `LangSmith`; `Datadog`; `AWS`; `Azure`. Source: [DHI Group](https://job-boards.greenhouse.io/dhigroupinc/jobs/8127959002) |

## Pairings That Look Real

| Pairing | Where seen | Read |
| --- | --- | --- |
| `MCP` + auth/policy/governance | Karbon AU; Forter UK; Smartsheet US; Omada US; DHI US | MCP is increasingly bundled with access control and enterprise safety requirements. |
| `LangGraph` + production workflow ownership | Bain AU; Awin UK; Dataiku UK; PlayStation US | LangGraph remains the clearest orchestration framework signal in app-layer roles. |
| AI gateway + eval + observability | Smartsheet US; Omada US | Gateway products are being paired with evals, PII controls, and telemetry rather than used as stand-alone infra. |
| `Python` + `TypeScript` dual-runtime ownership | Bain AU; StarRez AU; Smartsheet US; DHI US | Full-stack AI engineers are expected to own backend orchestration and product surfaces together. |
| graph retrieval + regulated reasoning | Catapult AU; Dataiku UK | Graph-heavy stacks remain niche but map to reasoning quality, explainability, and enterprise knowledge work. |
| Bedrock / Azure OpenAI + enterprise connectors | Dataiku UK; IFS UK; Smartsheet US; Karbon AU | Large-company roles still lean toward cloud-managed model access plus existing enterprise data systems. |
| training / serving infra + efficiency work | Firmus AU; Fireworks US; Assail US | Deep infra roles remain a minority, but when they appear they are highly specific and technically deep. |

## Language / Runtime Signal

| Runtime | Strength in sample | Typical shape |
| --- | --- | --- |
| `Python` | dominant | orchestration, evals, retrieval, backend AI systems, training/inference |
| `TypeScript` / `Node.js` | strong second | product AI, MCP servers, internal tools, workflow platforms |
| `SQL` | common but rarely foregrounded | retrieval, analytics, enterprise system access |
| `Go` / `Rust` | selective | high-performance systems and graph-heavy platform work |
| `C#` / `.NET` | selective | enterprise-integrated AI inside existing product stacks |
| `C++` / `CUDA` | niche but important | inference kernel and GPU-serving roles |

### Runtime read

- `Python` + `TypeScript` is still the most important pairing.
- `Node.js` keeps showing up where the role touches protocol servers, product UX, or enterprise integrations.
- GPU/system-level roles are still rare, but the US sample names their stack precisely when they exist.

## Safety / Robustness Signal

### Common

- evaluation frameworks
- regression suites
- tracing and observability
- guardrails
- structured outputs
- human review or operational ownership

### More explicit this week

- PII masking
- cost tracking
- audit logging
- RBAC
- `OAuth2`
- prompt-injection defenses
- jailbreak resistance
- policy/compliance controls
- governed model access through gateway layers

## Seen / Not Seen

### Seen clearly

- `MCP`
- `LangGraph`
- `OpenAI`
- `Anthropic`
- `Python`
- `TypeScript`
- `RAG`
- vector databases
- eval frameworks
- observability/tracing

### Seen, but not dominant

- `LiteLLM`
- `Kong AI Gateway`
- `OpenAI Agents SDK`
- `Claude Agent SDK`
- `Semantic Kernel`
- `GraphRAG`
- `AWS Bedrock`
- `Azure OpenAI`
- `Databricks`
- `Milvus`
- `Qdrant`
- `Vercel AI SDK`

### Not seen in this sample

- `Braintrust`
- `Promptfoo`
- `Langfuse`
- `Phoenix`
- `TensorZero`
- `Helicone`
- `FAISS`
- `turbopuffer`
- `Ollama`
- `GGUF`

## Regional Read

| Region | Strongest pattern | Distinctive signal this week |
| --- | --- | --- |
| Australia | split between full-stack agent product work and deep infra/training work | `Azure AI Search`/`Azure Container Apps` at Karbon and `TorchTitan`/`Megatron-LM`/`FSDP` at Firmus |
| United Kingdom | protocol-heavy applied AI plus enterprise connector stacks | `OpenAI Agents SDK` + `Claude Agent SDK` + `A2A` + `MCP` in one Dataiku role |
| United States | strongest control-plane and infra specificity | AI gateways, eval vendors, `Vercel AI SDK`, and explicit inference serving stacks |

## Source Set

- [Bain - Full Stack AI Product Engineer (DataEdge)](https://www.bain.com/careers/find-a-role/position/?jobid=106304)
- [Karbon - Senior AI Engineer](https://job-boards.greenhouse.io/karbon/jobs/6040484004)
- [StarRez - AI Engineer](https://job-boards.greenhouse.io/starrez/jobs/8165138002)
- [Caterpillar - Agentic AI / AI Ops Engineer](https://careers.caterpillar.com/en/jobs/r0000376964/agentic-ai-ai-ops-engineer-platform-engineering/)
- [Catapult Sports - Principal AI Engineer](https://job-boards.greenhouse.io/catapultsports/jobs/7902420)
- [Firmus - AI Engineer, AI & Applications](https://job-boards.greenhouse.io/firmus/jobs/5071736008)
- [Awin - Senior AI Engineer](https://job-boards.greenhouse.io/awin/jobs/7720582003)
- [Forter - Senior Applied AI Engineer](https://job-boards.greenhouse.io/forter/jobs/8486828002)
- [Dataiku - Applied AI Engineer](https://job-boards.greenhouse.io/dataiku/jobs/5978968004)
- [IFS - Forward Deployed AI Engineer](https://jobs.smartrecruiters.com/IFS1/744000128816717-forward-deployed-ai-engineer)
- [Smartsheet - Sr. Forward Deployed AI Engineer](https://job-boards.greenhouse.io/smartsheet/jobs/7930145)
- [PlayStation - Applied AI Engineer](https://boards.greenhouse.io/sonyinteractiveentertainmentglobal/jobs/5616918004)
- [Fireworks AI - AI Infrastructure Engineer](https://job-boards.greenhouse.io/fireworksai/jobs/4280748009)
- [Omada Health - Staff Software Engineer, AI Platform](https://job-boards.greenhouse.io/omadahealth/jobs/4738058008)
- [Assail - AI Engineer](https://jobs.ashbyhq.com/assail/7d92138f-7785-49df-a1d8-e757e84a4ebc)
- [DHI Group - Lead AI Engineer](https://job-boards.greenhouse.io/dhigroupinc/jobs/8127959002)
