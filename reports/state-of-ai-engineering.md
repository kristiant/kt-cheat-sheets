# State of AI Engineering

Date: 2026-06-29
Compared against prior run: 2026-06-23
Scope: sampled live AI Engineer / Applied AI Engineer / AI Product Engineer / Forward Deployed AI Engineer / ML Engineer postings across Australia, the US, and the UK.
Method: job postings as telemetry; concrete named stack only; official company pages preferred; board snippets used only where the local AU market is thinner.

## Week-over-Week Change

### New or materially clearer this week

| Signal | Evidence | Read |
| --- | --- | --- |
| `Google ADK` + `Claude API` + `Azure AI Foundry` + `Copilot Studio` in one enterprise role | Qcells US Applied AI Engineer | Multi-model / multi-vendor agent stacks are more explicit than last week, including Microsoft and Anthropic in the same implementation envelope. |
| `LangSmith` + `LangFuse` both named in a single role | Qcells US | Evals and tracing are showing up less as generic capability language and more as concrete vendor picks. |
| `DSPy` in forward-deployed AI work | Databricks UK AI Engineer / FDE | New this week. This is the first clear DSPy sighting in this report series. |
| Observability-native agent debugging | Datadog US Senior AI Engineer, APM Experiences | Clearer than last week: agents are being hired to explain traces, propose fixes, and generate monitoring artifacts directly from telemetry. |
| OpenAI field/deployment roles split by partner motion | OpenAI Sydney partner AI deployment role; OpenAI London forward deployed role | Deployment engineering is showing up as a distinct operating model: prototype fast, ship with customers, close product-feedback loops. |
| `FastMCP` on enterprise platforms | Qcells US | MCP implementation detail is getting more concrete than last week’s broader protocol-level mentions. |
| `A2A` / agent-to-agent interoperability language persists | Databricks UK; Veeam US in sampled live postings | Inter-agent protocol language is still niche, but it is no longer an isolated one-off. |

### Stronger than last week

- `MCP`
- Still the clearest protocol signal. This week it appears as `FastMCP`, internal servers, resource exposure, secure enterprise access, and forward-deployed integration work.
- `TypeScript` / `Node.js` in AI product roles
- Remains strong beside `Python`, especially where the role owns agents plus product surfaces, SDKs, or internal tools.
- observability as part of the AI product itself
- Datadog US and Future US reinforce that traces, metrics, logs, and replay are now part of the shipped AI workflow rather than background infra only.
- enterprise AI with explicit human escalation
- Qcells US and OpenAI deployment roles push on agent control boundaries, review flows, and support handoff instead of pure autonomy language.

### Weaker, thinner, or newly absent in this sample

- `MLflow`
- Still real in the broader market, but less visible in this week’s fresh sample than in last week’s enterprise-platform roles.
- graph-heavy retrieval
- `Neo4j`, `GraphRAG`, ontology-centric reasoning are not absent overall, but they are not the center of the current sample.
- deep training-infra stacks
- Last week’s strong AU infra/training signal (`TorchTitan`, `Megatron-LM`, `FSDP`, `DeepSpeed`) is not the center of this week’s live sample.
- `Promptfoo`, `DeepEval`, `Braintrust`, `Helicone`, `TensorZero`
- Not clearly seen in this week’s fresh postings.

### Trend call

The direction that looks more real now than it did on 2026-06-23: the market is getting more explicit about AI engineers owning the control plane around agents, not just prompts or model calls.

That control plane includes:

- eval/tracing vendors (`LangSmith`, `LangFuse`, `Datadog`, `OpenTelemetry`)
- protocol surfaces (`MCP`, `FastMCP`, `A2A`)
- enterprise control mechanisms (guardrails, escalation, access boundaries, secure data connectors)
- product-debugging loops built directly on telemetry

## Signal Map

### Most common in the sample

| Category | Stronger signals |
| --- | --- |
| Protocols / orchestration | `MCP`, `LangGraph`, tool calling, multi-agent workflows, human escalation loops |
| Model / inference layer | `OpenAI`, `Anthropic`, `Azure OpenAI`, `AWS Bedrock`, `Claude API` |
| Evals / observability | `LangSmith`, `LangFuse`, `OpenTelemetry`, `Datadog`, regression/evaluation suites |
| Retrieval / storage | vector search, semantic search, `pgvector`, enterprise data sources, graph DBs in a minority of roles |
| Languages / runtimes | `Python`, `TypeScript`, `Node.js`, `SQL`, `JavaScript`, some `Ruby`, `C#`, `Go`, `Rust` |
| Safety / robustness | guardrails, output validation, fallback flows, human review, prompt-injection awareness, auth and scoped access |

### Sharper niche signals

| Category | Signals |
| --- | --- |
| Enterprise agent control | `FastMCP`, `Copilot Studio`, escalation controls, automated QA, secure connectors |
| Forward-deployed stack | `DSPy`, `LangGraph`, `CrewAI`, `A2A`, rapid prototyping with direct customer integration |
| Observability-native AI | telemetry-to-agent workflows, SLO recommendation, trace analysis, root-cause assistance |
| Full-stack AI product engineering | `TypeScript` + `Python`; `Pydantic`; `JSON Schema`; `SSE`; webhooks; product-facing agent UX |

## Company -> Tool Matrix

### Australia

| Company | Role | Concrete signal |
| --- | --- | --- |
| OpenAI | Partner AI Deployment Engineer, AWS | `OpenAI API`; deployment engineering; rapid prototypes to production; customer-facing implementation; AWS-partner motion; safe/effective rollout language. Source: [OpenAI](https://openai.com/careers/partner-ai-deployment-engineer-aws-sydney-australia/) |
| OpenAI | Forward Deployed Engineer | rapid prototyping; software + product + GTM collaboration; production customer deployments; implementation feedback loop into core product. Source: [OpenAI](https://openai.com/careers/forward-deployed-engineer-sydney-australia/) |
| Karbon | Associate AI Engineer | `MCP`; `RAG`; `LangGraph`; `Agent SDK`; `ADK`; `Snowflake`; `dbt`; `Azure`; `Python`; `C#`; `sklearn`; `PyTorch`; `TensorFlow`; `spaCy`. Source: [Karbon](https://job-boards.greenhouse.io/karbon/jobs/5995013004) |
| Catapult Sports | Principal AI Engineer | `Go`; `Rust`; `Python`; `Neo4j`; `GraphRAG`; `AWS Bedrock`; `Promptfoo`; map-reduce over graph communities; `LLM-as-judge`; custom eval frameworks. Source: [Catapult Sports](https://job-boards.greenhouse.io/catapultsports/jobs/7902420) |

### United Kingdom

| Company | Role | Concrete signal |
| --- | --- | --- |
| OpenAI | Forward Deployed Software Engineer | customer-deployed AI systems; product iteration loop; fast implementation; cross-functional deployment engineering. Source: [OpenAI](https://openai.com/careers/forward-deployed-software-engineer-london-united-kingdom/) |
| Databricks | AI Engineer / FDE | `GenAI`; `MCP`; `A2A`; `LangGraph`; `AutoGen`; `CrewAI`; `DSPy`; customer-facing agent implementation on the Databricks platform. Source: [Databricks](https://www.databricks.com/company/careers/professional-services-operations/ai-engineer---fde-forward-deployed-engineer-8551531002) |
| Awin | Senior AI Engineer | `AWS`; `LangGraph`; deep agents; `LangSmith`; `MCP`; vector databases; hybrid retrieval; structured evals; regression testing; tracing/observability; caching/routing strategies. Source: [Awin](https://job-boards.greenhouse.io/awin/jobs/7720582003) |
| Forter | Senior Applied AI Engineer | `Claude Code`; `MCP`; `Strands Agents`; `RAG`; evaluation frameworks; transformation across Sales/Marketing/Finance/HR/Legal/CS. Source: [Forter](https://job-boards.greenhouse.io/forter/jobs/8486828002) |

### United States

| Company | Role | Concrete signal |
| --- | --- | --- |
| Qcells | Applied AI Engineer | `LangChain`; `Google ADK`; `FastMCP`; `Azure AI Foundry`; `Microsoft Copilot Studio`; `OpenAI API`; `Claude API`; `LangSmith`; `LangFuse`; vector databases; semantic retrieval; guardrails; output validation; rate limits; human escalation controls. Source: [Qcells](https://careers.qcells.com/jobs/17819541-applied-ai-engineer) |
| Datadog | Senior AI Engineer, APM Experiences | `Datadog`; `OpenTelemetry`; traces/metrics/logs; agentic debugging; AI-assisted incident analysis; SLO / dashboard / monitor generation; production observability-first AI. Source: [Datadog](https://careers.datadoghq.com/detail/7188589/) |
| Future | Applied AI Engineer | `LangChain`; `LangGraph`; tool/function calling; structured output; `Pydantic`; `JSON Schema`; `SSE`; webhooks; `AWS Bedrock`; `ECR`; `CloudFront`; `S3`; `Cognito`; `Langfuse`; `OpenTelemetry`; `Datadog`. Source: [Future](https://job-boards.greenhouse.io/future/jobs/4683133005) |
| ClickHouse | AI Product Engineer, ClickStack | `MCP`; agent SDKs; reusable skills; `TypeScript`; `Node.js`; `Python`; `SQL`; `Docker`; `Kubernetes`; `OpenTelemetry`; golden sets; `LLM-as-judge`; regression detection; prompt caching; context compaction. Source: [ClickHouse](https://job-boards.greenhouse.io/clickhouse/jobs/6014112004) |
| Carrot | Applied AI Engineer, Enterprise | `MCP`; Claude skills; plugins; `Claude Code`; `Python`; `TypeScript`; `JavaScript`; `Ruby`; `Workato`; `Salesforce`; `NetSuite`; `OAuth2`; webhooks; rate limiting; RBAC; audit logging; vector DBs; semantic search. Source: [Carrot](https://job-boards.greenhouse.io/carrotfertility/jobs/6011067004) |

## Tool Pairings That Look Real

| Pairing | Where seen | Read |
| --- | --- | --- |
| `MCP` + enterprise control surfaces | Qcells US; Carrot US; Karbon AU; Databricks UK | MCP is increasingly paired with secure connectors, scoped data access, plugins, and human-governed workflows. |
| `LangGraph` + eval/trace tooling | Awin UK; Future US; Karbon AU | LangGraph is often paired with explicit test/trace expectations, not just generic orchestration language. |
| `LangSmith` + `LangFuse` | Qcells US | Strong fresh signal that some teams are evaluating or using multiple observability/eval surfaces rather than standardizing on one by default. |
| observability stack + shipped AI workflow | Datadog US; Future US; ClickHouse US | Telemetry is part of the product loop itself: debugging, generation, regression detection, and incident assistance. |
| `Python` + `TypeScript` full-stack ownership | ClickHouse US; Carrot US; Future US; Awin UK | The strongest runtime pattern remains bilingual product engineering. |
| enterprise apps + agents | Carrot US; Qcells US | AI engineers are wiring agents into `Salesforce`, `NetSuite`, Microsoft enterprise tooling, and internal systems rather than isolating them as separate AI products. |

## Language / Runtime Signal

| Runtime | Strength in sample | Typical shape |
| --- | --- | --- |
| `Python` | dominant | orchestration, backend AI services, evals, retrieval, integration glue |
| `TypeScript` / `Node.js` | strong second | product-facing agents, SDKs, MCP servers, internal tools, workflow UIs |
| `SQL` | present but quieter this week | telemetry interrogation, governed retrieval, enterprise data access |
| `JavaScript` | common via product roles | adjacent to TypeScript in internal tools and product surfaces |
| `C#` / `Ruby` / `Go` / `Rust` | real but role-specific | enterprise integration, systems infra, performance-sensitive components |

### Runtime read

- `Python` + `TypeScript` is still the default pairing.
- `Node.js` is not just wrapper glue; it shows up where teams are shipping agent products and SDK-like surfaces.
- `SQL` remains important but this week’s sample leans more toward protocol/orchestration and observability than data-platform-heavy roles.

## Safety / Robustness Signal

### Common

- guardrails
- output validation
- regression / evaluation suites
- observability / tracing
- fallback and escalation flows
- secure enterprise access patterns

### More specific this week

- human escalation controls
- rate limiting at the workflow boundary
- prompt-injection / misuse awareness
- auth-aware tool access
- support / customer-review loops for production deployment

## Seen / Not Seen

### Seen clearly

- `MCP`
- `LangGraph`
- `OpenAI`
- `Anthropic` / `Claude API`
- `Python`
- `TypeScript`
- `RAG`
- tracing / observability
- structured outputs
- vector retrieval

### Seen, but not dominant

- `Google ADK`
- `LangChain`
- `LangSmith`
- `LangFuse`
- `AWS Bedrock`
- `Azure AI Foundry`
- `Copilot Studio`
- `CrewAI`
- `AutoGen`
- `DSPy`
- `GraphRAG`
- `Promptfoo`

### Not clearly seen in this sample

- `Braintrust`
- `DeepEval`
- `TensorZero`
- `Helicone`
- `Milvus`
- `Qdrant`
- `FAISS`
- `Semantic Kernel`
- `Vercel AI SDK`
- `OpenAI Agents SDK`
- `Claude Agent SDK`
- `vLLM`
- `TGI`
- `TensorRT-LLM`
- `Ollama`

## Regional Read

| Region | Strongest pattern | Distinctive signal this week |
| --- | --- | --- |
| Australia | deployment-oriented AI engineering plus a thinner but still real local agent-tooling layer | OpenAI deployment roles in Sydney; Karbon keeps AU tied to `MCP` / `LangGraph` / `ADK`; Catapult remains the clearest deeper eval + graph signal in-region |
| United Kingdom | forward-deployed and customer-embedded AI work | `DSPy`, `LangGraph`, `CrewAI`, `A2A`, `MCP`, and customer-facing implementation on platform vendors and internal transformation teams |
| United States | highest stack specificity and clearest observability / enterprise-control detail | `Google ADK` + `FastMCP` + `Azure AI Foundry` + `Copilot Studio`; `LangSmith` + `LangFuse`; observability-native agent roles at Datadog and ClickHouse |

## Source Set

- [OpenAI - Partner AI Deployment Engineer, AWS (Sydney)](https://openai.com/careers/partner-ai-deployment-engineer-aws-sydney-australia/)
- [OpenAI - Forward Deployed Engineer (Sydney)](https://openai.com/careers/forward-deployed-engineer-sydney-australia/)
- [Karbon - Associate AI Engineer](https://job-boards.greenhouse.io/karbon/jobs/5995013004)
- [Catapult Sports - Principal AI Engineer](https://job-boards.greenhouse.io/catapultsports/jobs/7902420)
- [OpenAI - Forward Deployed Software Engineer (London)](https://openai.com/careers/forward-deployed-software-engineer-london-united-kingdom/)
- [Databricks - AI Engineer / FDE](https://www.databricks.com/company/careers/professional-services-operations/ai-engineer---fde-forward-deployed-engineer-8551531002)
- [Awin - Senior AI Engineer](https://job-boards.greenhouse.io/awin/jobs/7720582003)
- [Forter - Senior Applied AI Engineer](https://job-boards.greenhouse.io/forter/jobs/8486828002)
- [Qcells - Applied AI Engineer](https://careers.qcells.com/jobs/17819541-applied-ai-engineer)
- [Datadog - Senior AI Engineer, APM Experiences](https://careers.datadoghq.com/detail/7188589/)
- [Future - Applied AI Engineer](https://job-boards.greenhouse.io/future/jobs/4683133005)
- [ClickHouse - AI Product Engineer, ClickStack](https://job-boards.greenhouse.io/clickhouse/jobs/6014112004)
- [Carrot - Applied AI Engineer, Enterprise](https://job-boards.greenhouse.io/carrotfertility/jobs/6011067004)
