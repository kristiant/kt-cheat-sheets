# State of AI Engineering

Date: 2026-06-23
Compared against prior run: 2026-06-22
Scope: sampled current AI Engineer / Applied AI Engineer / AI Product Engineer / ML Engineer / Agentic Engineer postings across Australia, the US, and the UK, using live company job pages.
Method: treat postings as telemetry; record named stack details only.

## Week-over-Week Change

### New or materially clearer this run

| Signal | Evidence | Read |
| --- | --- | --- |
| `OpenAI Agents SDK` | Dataiku UK; Okta US; PlayStation US | No longer a background mention. It now appears directly inside orchestration/framework lists for production roles. |
| `Claude Agent SDK` | Dataiku UK | Anthropic-adjacent agent SDK language is now explicit in a mainstream enterprise tooling role. |
| `Braintrust` | PhysicsX UK; ID.me US | Clearer than last run, and now paired with `OTel`, `LangSmith`, and eval platform design rather than generic observability. |
| `DSPy` | Databricks AU | New named framework signal in an Australia role, paired with `HuggingFace`, `LangChain`, RAG, multi-agent, and fine-tuning. |
| `ACP` | PhysicsX UK | First explicit appearance in the sampled postings alongside `MCP` and `A2A`. |
| agent security standards bundle | Okta US | `OWASP Top 10 for Agentic Applications`, `NIST AI RMF`, `MITRE ATLAS`, `ISO/IEC 42001`, `EU AI Act` appear in one concrete stack. |
| fine-grained auth stack for agents | Okta US | `RFC 8693`, `DPoP`, `SCIM`, `ReBAC`, `ABAC`, `OPA`, `Cedar`, `OpenFGA` are now directly attached to agent engineering work. |
| voice-agent stack | Zoom AU | `ASR`, `TTS`, turn-taking, barge-in, `ElevenLabs`, `Azure`, `Cartesia`, plus contact-center vendors make voice deployment look more operational than last week. |
| inference serving stack | Anaplan UK | `vLLM`, `TensorRT`, `Ray` surfaced together in a general enterprise AI engineer role rather than a pure infra role. |
| workflow / app-layer tools around agents | PlayStation US | `N8N`, `AWS Bedrock Agents`, `Knowledge Bases`, `OpenAI Agents SDK` show up in the same orchestration set. |

### Stronger than last run

- `MCP`
  - Still broad, but more roles now expect building or publishing MCP servers, not just consuming them.
- evals as a product/platform layer
  - `ID.me`, `PhysicsX`, `Dataiku`, `Okta`, `PlayStation`, and `Anaplan` all talk about evals, monitoring, regressions, or observability with concrete technical ownership.
- TypeScript as an AI runtime
  - `Node.js` / `TypeScript` shows up in `Zoom`, `Dataiku`, `PhysicsX`, `PlayStation`, and the prior `Bain` / `Ripple` / `Smartsheet` sample. This is not just frontend adjacency.
- governance and safe delegation
  - Security language is more technical this run: token exchange, policy engines, rogue-agent detection, sandboxing, kill switches, scope sprawl, and delegation-chain monitoring.

### Weaker, thinner, or absent in this run

- `Strands` / `AgentCore`
  - Very strong in the previous sample, but not central in this fresh set.
- `Langfuse`
  - Present last run; weaker this run versus `LangSmith`, `OTel`, `Braintrust`, and generic eval infra.
- graph-heavy retrieval
  - `GraphRAG` appears in `Dataiku`, but `Neo4j` / graph-community pipelines are not as visible as in the prior AU sample.
- `Promptfoo`
  - Not clearly present in the new sample.
- `Qdrant`
  - Not visible in this set, while `Pinecone`, `Weaviate`, `pgvector`, `OpenSearch`, and Azure-native search patterns remain more commonly named.

### Direction call

The trend that looks more real now: the market is hardening around agent platform engineering, not just application prototyping. The clearest hiring signal is the combination of protocol work (`MCP`, `A2A`, `ACP`), eval pipelines, identity/policy controls, and typed service/runtime engineering around agents.

## Signal Map

### Strongest signals in this sample

| Category | Stronger signals | Notes |
| --- | --- | --- |
| Protocols / orchestration | `MCP`, `LangGraph`, `A2A`, `OpenAI Agents SDK` | `MCP` remains the center of gravity; SDK/protocol choice is becoming explicit. |
| Eval / observability | `LangSmith`, `OpenTelemetry`, `Braintrust`, regression suites, drift monitoring, LLM-as-judge | Evals are increasingly an owned platform concern. |
| Clouds / model platforms | `AWS`, `Azure`, `GCP`, `AWS Bedrock`, `OpenAI`, `Anthropic`, `Gemini` | Multi-provider routing is now a repeated requirement, not a nice-to-have. |
| Languages / runtimes | `Python`, `TypeScript`, `Node.js`, `Go`, `Java` | `Python` + `TypeScript` remains the dominant pairing. |
| Retrieval / storage | `Pinecone`, `Weaviate`, `OpenSearch`, `pgvector`, `Redis`, `Azure AI Search`, `Knowledge Banks`, vector + hybrid retrieval | Still more common than one single "winner" DB. |
| Safety / robustness | guardrails, red-teaming, prompt injection mitigation, sandboxing, policy adherence, cost tracking, latency tracking | Security/quality language is much more operational than generic. |

### Sharper niche signals

| Category | Signals |
| --- | --- |
| Protocol standards | `MCP`, `A2A`, `ACP` |
| Agent identity / auth | `OAuth 2.0`, `OIDC`, `SAML`, `SCIM`, `RFC 8693`, `DPoP`, `ReBAC`, `ABAC`, `OPA`, `Cedar`, `OpenFGA` |
| Eval vendors / stacks | `Braintrust`, `LangSmith`, `Arize Phoenix`, `OpenTelemetry`, `Datadog`, `W&B`, `MLflow` |
| Inference / serving | `vLLM`, `TensorRT`, `Ray` |
| Voice agent runtime | `ASR`, `TTS`, turn-taking, barge-in, `ElevenLabs`, `Cartesia` |
| App / workflow layer | `N8N`, `SWR`, `FastAPI`, `AWS Lambda`, `Vue.js`, `Node.js` |

## Company -> Signal Matrix

### Australia

| Company | Role | Concrete signal |
| --- | --- | --- |
| Zoom | Applied AI Engineer | voice agents; tool use; multi-step orchestration; guardrails; `ASR`; `TTS`; turn-taking; barge-in; `ElevenLabs`; `Azure`; `Cartesia`; `Python`; working knowledge of `TypeScript` or `Go`; `Genesys`; `Five9`; `NICE`; `Zoom Contact Center`; `Decagon`; `Sierra`. Source: [Zoom](https://careers.zoom.us/jobs/applied-ai-engineer-remote-australia) |
| Databricks | AI Engineer - FDE | `Mosaic AI`; `RAG`; multi-agent systems; `Text2SQL`; fine-tuning; `HuggingFace`; `LangChain`; `DSPy`; evaluation and optimizations; `pandas`; `scikit-learn`; `PyTorch`; `AWS`; `Azure`; `GCP`; `Apache Spark`. Source: [Databricks](https://www.databricks.com/company/careers/professional-services-operations/ai-engineer---fde-forward-deployed-engineer-8298792002) |
| Bain (`DataEdge`) | Full Stack AI Product Engineer | `TypeScript`; `Node.js`; `Python`; `LangGraph`; `FastAPI`; `Pydantic v2`; `Temporal`; `pgvector`; `OpenSearch`; `Docker`; `Kubernetes`; `tRPC`; `Zod`; `OpenAPI codegen`; `Vitest`; `Jest`; `Vite`; `Turbopack`; `esbuild`. Source: [Bain](https://www.bain.com/careers/find-a-role/position/?jobid=106304) |
| Karbon | Associate AI Engineer | `RAG`; chaining; `MCP`; `Python`; `sklearn`; `PyTorch`; `TensorFlow`; `spaCy`; `ADK`; `LangGraph`; `Agent SDK`; `Snowflake`; `dbt`; `C#`; `Azure`. Source: [Karbon](https://job-boards.greenhouse.io/karbon/jobs/5995013004) |

### United Kingdom

| Company | Role | Concrete signal |
| --- | --- | --- |
| Dataiku | Generative AI Engineer | `Dataiku Agent Hub`; `Prompt Studio`; `LLM Mesh`; `Knowledge Banks`; `LangGraph`; `CrewAI`; `Claude Agent SDK`; `OpenAI Agents SDK`; `OpenAI`; `Anthropic`; `Gemini`; `AWS Bedrock`; `Azure`; open-source models; `GraphRAG`; reranking; dynamic filtering; custom `MCP` servers; evals; `A2A`; `Cost Guard`; `Quality Guard`; `Vue.js`; `Node.js`; `Claude Code`; `Cursor`. Source: [Dataiku](https://job-boards.greenhouse.io/dataiku/jobs/5978968004) |
| PhysicsX | Principal Agentic Engineer | `MCP`; `A2A`; `ACP`; durable execution; agent memory; `Python`; `Go`; `TypeScript`; `Kubernetes`; `Docker`; `Terraform`; `LangGraph`; `LangChain`; `Temporal`; `Pinecone`; `Weaviate`; `OTel`; `LangSmith`; `Arize`; `Braintrust`; token cost tracking; agent sandboxing; permissioning; `PydanticAI`. Source: [PhysicsX](https://job-boards.eu.greenhouse.io/physicsx/jobs/4804769101) |
| Anaplan | Senior AI Engineer | GenAI + ML systems; backend APIs; RAG; conversational interfaces; agentic workflows; fine-tuning; `Python`; vector DBs (`Pinecone`, `Weaviate`, `Qdrant`); model serving (`vLLM`, `TensorRT`, `Ray`); observability (`LangSmith`, `W&B`, `MLflow`); A/B testing; inference cost and scalability optimization. Source: [Anaplan](https://job-boards.greenhouse.io/anaplan/jobs/8573871002?ref=employbl) |
| Baringa | Senior Consultant, Forward Deployed AI Engineer | major LLM SDKs; agent frameworks; `RAG`; `MCP` servers; `scikit-learn`; `PyTorch`; `XGBoost`; `SageMaker`; `Azure ML`; `Vertex AI`; `FastAPI`; `AWS Lambda`; event-driven patterns; `React`; `Next.js`; `Material UI`; `SWR`; `Jest`; `Cypress`; `React Test Library`. Source: [Baringa](https://job-boards.eu.greenhouse.io/baringa/jobs/4869804101) |

### United States

| Company | Role | Concrete signal |
| --- | --- | --- |
| Okta | Senior Forward Deployed Engineer - Okta for AI Agents | `OWASP Top 10 for Agentic Applications`; `NIST AI RMF`; `MITRE ATLAS`; `HIPAA`; `FedRAMP`; `SOC 2`; `OAuth 2.0`; `OIDC`; `SAML`; `SCIM`; `RFC 8693`; `DPoP`; `MCP`; `A2A`; `ISO/IEC 42001`; `EU AI Act`; `ReBAC`; `ABAC`; `OPA`; `Cedar`; `OpenFGA`; `Claude`; `ChatGPT`; `Microsoft Copilot`; `Agentforce`; `Bedrock`; `LangChain`; `CrewAI`; `OpenAI Agents SDK`; `MCP` servers; `Claude Code`; `Cursor`; rogue-agent detection; kill-switch verification; anomalous delegation chains. Source: [Okta](https://www.okta.com/company/careers/okta-for-ai-agents/senior-forward-deployed-engineer-okta-for-ai-agents-7961356/) |
| ID.me | Staff Software Engineer - AI Agent Evaluations | `RAG`; data chunking; pre-retrieval data quality gates; vector search accuracy; hallucination reduction; eval pipelines; drift monitoring; LLM-as-judge; property-based testing; golden datasets; `Claude Code`; `Cursor`; `Python`; `Java`; `Go`; `Braintrust`; `LangChain`; `LangGraph`; `CrewAI`; human-in-the-loop evals; shadow scoring; red-teaming; `Datadog`; `OpenTelemetry`. Source: [ID.me](https://job-boards.greenhouse.io/idme/jobs/7766372003) |
| PlayStation | AI Engineer | `AWS`; async processing; RAG; chunking; indexing; reranking; citations; vector and hybrid search; `OpenSearch`; `Pinecone`; `Weaviate`; `Redis`; `pgvector`; `Azure AI Search`; orchestration via `LangChain`; `LangGraph`; `LlamaIndex`; `Semantic Kernel`; `OpenAI Agents SDK`; `N8N`; `AWS Bedrock Agents`; `Knowledge Bases`; observability via `LangSmith`; `Arize Phoenix`; `OpenTelemetry`; `Datadog`; `Splunk`; `New Relic`; `CloudWatch`; `MCP`. Source: [PlayStation](https://job-boards.greenhouse.io/sonyinteractiveentertainmentglobal/jobs/6007946004) |
| Pairwise | Network Infrastructure & AI Engineer | dual-stack infra + AI role; `Azure`; `AWS`; VNet/VPC; private endpoints; `Terraform`; `Bicep`; `Kubernetes`; model gateways; vector stores; ingestion pipelines; `LangGraph`; `LangChain`; RAG; evals-as-CI; tracing; cost and latency monitoring; `Python`. Source: [Pairwise](https://job-boards.greenhouse.io/pairwiseviadelart/jobs/4694763005) |

## Tool -> Companies

| Tool / platform | Companies |
| --- | --- |
| `MCP` | Karbon, Dataiku, PhysicsX, Baringa, Okta, PlayStation |
| `A2A` | Dataiku, PhysicsX, Okta |
| `ACP` | PhysicsX |
| `LangGraph` | Bain, Karbon, Dataiku, PhysicsX, PlayStation, Pairwise |
| `LangChain` | Databricks, Dataiku, PhysicsX, PlayStation, Pairwise, Baringa, Okta |
| `OpenAI Agents SDK` | Dataiku, Okta, PlayStation |
| `Claude Agent SDK` | Dataiku |
| `CrewAI` | Dataiku, Okta, ID.me |
| `PydanticAI` | PhysicsX |
| `DSPy` | Databricks |
| `Temporal` | Bain, PhysicsX |
| `Braintrust` | PhysicsX, ID.me |
| `LangSmith` | PhysicsX, Anaplan, PlayStation |
| `Arize` / `Phoenix` | PhysicsX, PlayStation |
| `OpenTelemetry` | PhysicsX, ID.me, PlayStation |
| `MLflow` | Anaplan |
| `W&B` | Anaplan |
| `Pinecone` | PhysicsX, Anaplan, PlayStation |
| `Weaviate` | PhysicsX, Anaplan, PlayStation |
| `Qdrant` | Anaplan |
| `pgvector` | Bain, PlayStation |
| `OpenSearch` | Bain, PlayStation |
| `Azure AI Search` | PlayStation |
| `AWS Bedrock` | Dataiku, Okta, PlayStation |
| `vLLM` | Anaplan |
| `TensorRT` | Anaplan |
| `Ray` | Anaplan |
| `HuggingFace` | Databricks |
| `ElevenLabs` | Zoom |
| `Cartesia` | Zoom |
| `Claude Code` | Dataiku, Okta, ID.me |
| `Cursor` | Dataiku, Okta, ID.me |

## Pairings That Look Real

| Pairing | Where seen | Why it matters |
| --- | --- | --- |
| `MCP` + `A2A` + agent SDKs | Dataiku UK; Okta US; PhysicsX UK | Protocol layering is becoming explicit: tool access, inter-agent communication, and application-owned orchestration are being separated. |
| `MCP` + auth/policy engines | Okta US | Agent plumbing is now directly tied to enterprise authorization architecture, not treated as an afterthought. |
| `LangGraph` + durable execution | Bain AU; PhysicsX UK | Durable workflow thinking is sticking to agent platform roles, not just backend workflow teams. |
| `Python` + `TypeScript` + full-stack AI product ownership | Zoom AU; Dataiku UK; PhysicsX UK; Bain AU | Product AI roles increasingly span backend agents, orchestration, thin UIs, and typed integration layers. |
| `Braintrust` + `LangSmith` + `OTel` | PhysicsX UK | Teams are mixing vendor-specific AI eval tooling with standard telemetry rather than choosing one or the other. |
| `OpenAI Agents SDK` + `LangGraph` + `MCP` | Dataiku UK; PlayStation US | The market is not converging on one framework; teams are composing stacks. |
| voice AI + contact-center stack + agent platforms | Zoom AU | Voice AI is showing up as production deployment engineering, not novelty assistant work. |
| vector DB plurality + inference serving | Anaplan UK; PlayStation US | Retrieval choice and serving choice are both open design dimensions; no single default stack. |
| infra networking + agent platform | Pairwise US | Some AI engineer roles are explicitly hybridizing cloud/network engineering with RAG and agent ops. |

## Language / Runtime Signal

| Runtime | Strength in sample | Typical role shape |
| --- | --- | --- |
| `Python` | dominant | orchestration, evals, backend AI services, agent APIs |
| `TypeScript` / `Node.js` | strong | agent tools, webapps, platform integration layers, product surfaces |
| `Go` | moderate but high-signal | systems-heavy platform roles, especially around agent infrastructure |
| `Java` | present but secondary | enterprise service integration and backend support roles |
| frontend runtimes | steady | `Vue.js`, `React`, `Next.js`, `SWR`, `N8N` appear as part of agent product delivery, not separate workstreams |

### Runtime read

- The most common real-world shape remains `Python` plus a typed application/runtime layer (`TypeScript` / `Node.js`).
- `Go` remains a sharp signal when the team owns platform internals or secure execution paths.
- The "AI engineer" title now regularly includes frontend or full-stack delivery requirements when the system is agent-facing or tool-heavy.

## Safety / Robustness Signal

### Common

- guardrails
- eval frameworks
- regression suites
- monitoring and observability
- prompt engineering with validation
- cost / latency tracking
- secure enterprise deployment

### More specific this run

- red-teaming
- property-based testing for AI outputs
- golden datasets
- shadow scoring
- policy adherence checks
- rogue-agent detection
- anomalous delegation-chain monitoring
- kill-switch verification
- sandboxing and permissioning
- prompt injection / context leakage mitigation
- `OWASP Top 10 for Agentic Applications`
- `NIST AI RMF`
- `MITRE ATLAS`
- `ISO/IEC 42001`
- `EU AI Act`

## Seen / Not Seen

### Seen clearly

- `MCP`
- `LangGraph`
- `Python`
- `TypeScript`
- `AWS`
- `Azure`
- `OpenAI`
- `Anthropic`
- eval pipelines
- observability
- vector retrieval
- cost / latency optimization
- `Claude Code`
- `Cursor`

### Seen, but not dominant

- `OpenAI Agents SDK`
- `Claude Agent SDK`
- `CrewAI`
- `PydanticAI`
- `DSPy`
- `Braintrust`
- `vLLM`
- `TensorRT`
- `Ray`
- `N8N`
- `Semantic Kernel`
- `GraphRAG`

### Not clearly seen in this sample

- `Langfuse`
- `Promptfoo`
- `TensorZero`
- `Helicone`
- `Milvus`
- `turbopuffer`
- `AutoGen`
- `Semantic Kernel` as a primary default
- `Strands` / `AgentCore` as a central stack

## Regional Read

| Region | Strongest pattern | Distinctive signal this run |
| --- | --- | --- |
| Australia | customer-facing deployment and full-stack AI product engineering | voice-agent operations (`Zoom`) and `DSPy` + `Mosaic AI` (`Databricks`) |
| United Kingdom | protocol-rich enterprise agent platform work | `OpenAI Agents SDK`, `Claude Agent SDK`, `MCP`, `A2A`, `ACP`, `Braintrust`, `Temporal`, `vLLM`, `TensorRT` |
| United States | identity, policy, and evaluation-heavy agent engineering | fine-grained auth for agents (`Okta`), eval platform design (`ID.me`), broad orchestration/search stack (`PlayStation`) |

## Source Set

- [Zoom - Applied AI Engineer (Australia)](https://careers.zoom.us/jobs/applied-ai-engineer-remote-australia)
- [Databricks - AI Engineer - FDE (Sydney)](https://www.databricks.com/company/careers/professional-services-operations/ai-engineer---fde-forward-deployed-engineer-8298792002)
- [Bain - Full Stack AI Product Engineer (DataEdge)](https://www.bain.com/careers/find-a-role/position/?jobid=106304)
- [Karbon - Associate AI Engineer](https://job-boards.greenhouse.io/karbon/jobs/5995013004)
- [Dataiku - Generative AI Engineer (UK)](https://job-boards.greenhouse.io/dataiku/jobs/5978968004)
- [PhysicsX - Principal Agentic Engineer (UK)](https://job-boards.eu.greenhouse.io/physicsx/jobs/4804769101)
- [Anaplan - Senior AI Engineer (UK)](https://job-boards.greenhouse.io/anaplan/jobs/8573871002?ref=employbl)
- [Baringa - Senior Consultant, Forward Deployed AI Engineer (UK)](https://job-boards.eu.greenhouse.io/baringa/jobs/4869804101)
- [Okta - Senior Forward Deployed Engineer, Okta for AI Agents (US)](https://www.okta.com/company/careers/okta-for-ai-agents/senior-forward-deployed-engineer-okta-for-ai-agents-7961356/)
- [ID.me - Staff Software Engineer, AI Agent Evaluations (US)](https://job-boards.greenhouse.io/idme/jobs/7766372003)
- [PlayStation - AI Engineer (US)](https://job-boards.greenhouse.io/sonyinteractiveentertainmentglobal/jobs/6007946004)
- [Pairwise - Network Infrastructure & AI Engineer (US)](https://job-boards.greenhouse.io/pairwiseviadelart/jobs/4694763005)
