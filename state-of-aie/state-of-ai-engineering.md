# State of AI Engineering

Date: 2026-06-23
Compared against prior run: 2026-06-22
Scope: sampled current AI Engineer / Applied AI Engineer / AI Product Engineer / ML Engineer / Forward Deployed AI Engineer postings across Australia, the US, and the UK.
Method: treat postings as telemetry; record concrete named stack signals visible on official job pages on 2026-06-23; prefer company career pages over aggregators.

## Week-over-Week Change

### New or materially clearer this week

| Signal | Evidence | Read |
| --- | --- | --- |
| `ClickHouse` as agent substrate | ClickHouse US AI Product Engineer | New concrete pattern: observability platform vendors are now hiring for agent layers directly on top of logs, metrics, traces, and session replay. |
| `skills` as an explicit engineering primitive | ClickHouse US; Carrot US | Last week had agent/tool language; this week has direct hiring language for reusable skills, Claude skills, and Markdown-encoded workflows. |
| `Claude skills` + `Claude Code` + `MCP` in one role | Carrot US | Stronger developer-agent stack coherence than last week. This is not generic "AI familiarity"; it is operationalized into build environment, skill packaging, and governed tool access. |
| `Workato` / iPaaS + agent infrastructure | Carrot US | New product-engineering pattern: internal enterprise AI roles are mixing MCP servers with integration platforms instead of building everything as bespoke services. |
| `Unity Catalog` + `MLflow` + `Databricks` + `MCP gateways` | Exactera US | Sharper enterprise AI platform stack than last week, with governance and agent interface layers named together. |
| `knowledge graph` / `domain ontology` / `versioned graph snapshots` | Exactera US | Graph-backed, audit-defensible reasoning is more explicit this week than the lighter GraphRAG signal seen last week. |
| `prompt caching` / `context compaction` | ClickHouse US | Inference efficiency language is becoming more operational and less theoretical. |
| distributed training stack depth | Firmus AU | New Australia-side signal: `TorchTitan`, `Megatron-LM`, `FSDP`, `DeepSpeed`, `NCCL`, `Slurm`, `LoRA`, `QLoRA`, `JAX` show up together in one production role. |
| `AI Ops` as agent-platform practice | Caterpillar AU | New AU signal: `LangGraph` and `AutoGen` are framed as platform-operations tooling, not only product UX tooling. |
| `LLM-as-judge` wired into `CI/CD` | Cadence US | Stronger eval operationalization in regulated workflows than last week’s more generic eval-language. |

### Stronger than last week

- `MCP`
- This remains the clearest cross-region protocol signal. This week it showed up as internal MCP servers, MCP gateways, MCP server/client development, MCP data-access layers, and MCP auth/scoping design.
- internal business transformation roles using agent stacks
- Forter UK, Carrot US, and DeepIntent US all describe engineers embedded with non-R&D or business teams, building production AI systems directly into operational workflows.
- compliance-grade / audit-grade AI language
- Exactera US and Carrot US push further than generic "guardrails" into replayability, audit logging, role-based access control, governed data access, and deterministic outcomes.
- `TypeScript` as AI runtime, not accessory language
- ClickHouse US, Carrot US, Bain AU, Smartsheet UK, and last week’s platform roles continue to reinforce `TypeScript`/`Node.js` as first-class in agent systems, SDKs, and platform interfaces.

### Weaker, thinner, or newly absent in this sample

- `Strands` / `AgentCore`
- Still present in the carried-forward sample from prior week, but not the dominant new-signal center this week.
- `Langfuse`
- Still real, but fewer fresh postings this week used it as the lead observability anchor.
- `Phoenix`
- Not prominent in the fresh sampled set.
- `Promptfoo`
- No new sightings in this week’s additions.
- `DeepEval`, `TensorZero`, `Helicone`, `turbopuffer`
- Not seen in this sample.

### Trend call

The direction that looks more real now than it did last week: AI engineering roles are splitting into two sharper modes.

1. Agent platform / observability / protocol roles
2. Embedded business-systems / workflow-transformation roles

Both modes still use `MCP`, `RAG`, evals, and tool calling, but the surrounding stack is diverging. One side is moving toward tracing, SDKs, context engineering, and telemetry substrates. The other is moving toward governed data access, RBAC, OAuth, iPaaS, enterprise systems, and business-process automation.

## Signal Map

### Strongest signals in the sample

| Category | Stronger signals | Read |
| --- | --- | --- |
| Protocols / orchestration | `MCP`, `LangGraph`, tool calling, multi-step workflows | `MCP` is still the center of gravity. |
| Model / API layer | `Anthropic`, `OpenAI`, `AWS Bedrock`, open-source models | Closed-model APIs remain dominant; open-source appears more in infra-heavy roles. |
| Eval / observability | eval frameworks, regression suites, `LLM-as-judge`, tracing, `OpenTelemetry` | Capability language is still more common than one-vendor-only language. |
| Retrieval / memory | `RAG`, vector stores, hybrid retrieval, knowledge graphs, graph traversal, structured queries | Retrieval stacks are getting more differentiated. |
| Languages / runtimes | `Python`, `TypeScript`/`Node.js`, `SQL`, `Go` | `Python` + `TypeScript` remains the default bilingual pairing. |
| Safety / robustness | guardrails, audit logging, RBAC, `OAuth2`, structured outputs, replayability | Governance is getting more concrete. |

### Sharper niche signals

| Category | Signals |
| --- | --- |
| Distributed training / model efficiency | `TorchTitan`, `Megatron-LM`, `FSDP`, `DeepSpeed`, `NCCL`, `Slurm`, `LoRA`, `QLoRA`, `JAX` |
| Compliance-grade reasoning | domain ontology, knowledge graph, versioned graph snapshots, versioned prompts, versioned model snapshots |
| Observability-native agents | `ClickHouse`, `OpenTelemetry`, traces, session replay, incident investigation agents |
| Enterprise AI integration | `Workato`, `Salesforce`, `NetSuite`, `REST APIs`, `OAuth2`, webhooks, rate limiting |
| Agent developer packaging | skills, Claude skills, plugins, SDKs, MCP servers, prompt libraries |

## Company -> Tool Matrix

### Australia

| Company | Role | Concrete signal |
| --- | --- | --- |
| Firmus Technologies | AI Engineer, AI & Applications | `TorchTitan`; `Megatron-LM`; `FSDP`; tensor/pipeline parallelism; `NCCL`; `PyTorch`; `JAX`; `DeepSpeed`; `K8s`; `Slurm`; offline evaluation harnesses; `LoRA`; `QLoRA`. Source: [Firmus](https://job-boards.greenhouse.io/firmus/jobs/5071736008) |
| Caterpillar | Agentic AI / AI Ops Engineer | `Kubernetes`; observability logs/metrics/traces; telemetry pipelines; `LangGraph`; `AutoGen`; multi-step workflows; tool integration; AI Ops / SRE / reliability framing. Source: [Caterpillar](https://careers.caterpillar.com/en/jobs/r0000376964/agentic-ai-ai-ops-engineer-platform-engineering/) |
| Bain (`DataEdge`) | Full Stack AI Product Engineer | `TypeScript`; `Node.js`; `Python`; `LangGraph`; `FastAPI`; `Pydantic v2`; `Temporal`; `pgvector`; `OpenSearch`; `Docker`; `Kubernetes`; `tRPC`; `Zod`; `OpenAPI codegen`. Source: [Bain](https://www.bain.com/careers/find-a-role/position/?jobid=106304) |
| Karbon | Associate AI Engineer | `MCP`; `RAG`; `LangGraph`; `Agent SDK`; `ADK`; `Snowflake`; `dbt`; `Azure`; `Python`; `C#`; `sklearn`; `PyTorch`; `TensorFlow`; `spaCy`. Source: [Karbon](https://job-boards.greenhouse.io/karbon/jobs/5995013004) |
| Catapult Sports | Principal AI Engineer | `Go`; `Rust`; `Python`; `Neo4j`; `GraphRAG`; `AWS Bedrock`; `Promptfoo`; map-reduce over graph communities; `LLM-as-judge`; custom eval frameworks. Source: [Catapult Sports](https://job-boards.greenhouse.io/catapultsports/jobs/7902420) |

### United Kingdom

| Company | Role | Concrete signal |
| --- | --- | --- |
| Smartsheet | Sr. Forward Deployed AI Engineer | multi-agent orchestration; `MCP` resource packs; `Python`; `TypeScript`/`JavaScript`; deployment kits; `AWS Bedrock`; `Temporal`; `Databricks`; `Strands SDK`; `LangChain`; open-source MCP servers. Source: [Smartsheet](https://job-boards.greenhouse.io/smartsheet/jobs/7930145) |
| Awin | Senior AI Engineer | `AWS`; `LangGraph`; `Deep Agents`; `LangSmith`; `MCP`; vector databases; hybrid retrieval; structured evals; regression tests; tracing/observability; caching and routing strategies. Source: [Awin](https://job-boards.greenhouse.io/awin/jobs/7720582003) |
| Forter | Senior Software AI Engineer | internal `MCP` server; `RAG`; evaluation frameworks; AI guardrails; reusable models/tools/APIs/platforms; `Claude Code`; `Cursor`; `GitHub Copilot`; cloud on `AWS` / `GCP` / `Azure`. Source: [Forter](https://job-boards.greenhouse.io/forter/jobs/8415897002) |
| Forter | Senior Applied AI Engineer | `Claude Code`; `MCP`; `Strands Agents`; `RAG`; evaluation frameworks; AI transformation inside Sales/Marketing/Finance/HR/Legal/CS; adoption measurement. Source: [Forter](https://job-boards.greenhouse.io/forter/jobs/8486828002) |
| Palantir | Forward Deployed AI Engineer | production LLM workflows; data processing pipelines; advanced analytics tools; `Python`; `Java`; `C++`; `TypeScript`/`JavaScript`; customer-embedded AI delivery. Source: [Palantir](https://jobs.lever.co/palantir/ff1029bd-bb6d-4d78-a03e-5f9744d0b798) |
| IFS | Forward Deployed AI Engineer | `OpenAI`; `Hugging Face`; `LangChain`; `Pinecone`; `Weaviate`; `Salesforce`; `ServiceNow`; `SAP`; `Oracle`; enterprise AI platform + multi-agent orchestration language. Source: [IFS](https://jobs.smartrecruiters.com/IFS1/744000128816717-forward-deployed-ai-engineer) |

### United States

| Company | Role | Concrete signal |
| --- | --- | --- |
| ClickHouse | AI Product Engineer - ClickStack | `MCP`; agent SDKs; reusable skills; `TypeScript`/`Node.js`; `Python`; `SQL`; `Docker`; `Kubernetes`; `OpenTelemetry`; golden sets; `LLM-as-judge`; regression detection; prompt caching; context compaction; observability-native agents. Source: [ClickHouse](https://job-boards.greenhouse.io/clickhouse/jobs/6014112004) |
| Exactera | Principal AI Engineer | `Databricks`; `Unity Catalog`; `MLflow`; `Anthropic API`; `OpenAI API`; `MCP`; production MCP gateways; `AWS`; `Terraform`; knowledge graph; domain ontology; hybrid retrieval; graph traversal; versioned graph snapshots; prompt caching; Python and Node SDKs. Source: [Exactera](https://job-boards.greenhouse.io/exactera/jobs/7740360003) |
| Carrot | Applied AI Engineer, Enterprise | `MCP` architecture; Claude skills; plugins; `Claude Code`; `Python`; `TypeScript`/`JavaScript`; `Ruby`; `Workato`; `Salesforce`; `NetSuite`; `OAuth2`; webhooks; rate limiting; HIPAA; RBAC; audit logging; vector databases; semantic search. Source: [Carrot](https://job-boards.greenhouse.io/carrotfertility/jobs/6011067004) |
| Cadence | Applied AI Engineer | `OpenAI`; `Anthropic`; open-source models; tool use / function calling; structured outputs; vector stores; `RAG`; offline benchmarks; safety tests; regression suites; `LLM-as-judge`; `CI/CD`; multi-agent coordination; `SFT`; `RLHF`; `LoRA`; HIPAA; `SOC 2`. Source: [Cadence](https://www.cadence.care/jobs/applied-ai-engineer/) |
| DeepIntent | Applied AI Engineer | `OpenAI`; `Anthropic Claude`; `Claude Code`; `MCP` servers; `Python`; REST APIs; workflow automation tools; business-embedded agentic operating system language. Source: [DeepIntent](https://job-boards.greenhouse.io/deepintent/jobs/5979034004) |
| Addepar | Sr. Software Engineer - AI/ML - AI Platform | `Python`; `Golang`; `gRPC`; `AWS`; `Kubernetes`; vector DBs; agentic frameworks; `MCP` / cross-platform tool use; web browsing; computer use. Source: [Addepar](https://job-boards.greenhouse.io/addepar1/jobs/8479770002) |
| Capstone Investment Advisors | AI Engineer | `Python`; `LangChain`; `LlamaIndex`; vector stores; `OpenAI API`; `Anthropic API`; MCP servers/clients; `RAG`; function calling. Source: [Capstone](https://job-boards.greenhouse.io/capstoneinvestmentadvisors/jobs/8453280002) |
| Future | Applied AI Engineer | `LangChain`; `LangGraph`; tool/function calling; structured output; `Pydantic`; `JSON Schema`; `SSE`; webhooks; `AWS Bedrock`; `ECR`; `CloudFront`; `S3`; `Cognito`; `Langfuse`; `OpenTelemetry`; `Datadog`. Source: [Future](https://job-boards.greenhouse.io/future/jobs/4683133005) |
| Veeam | Staff AI Engineer | `Azure OpenAI Service`; `Kubernetes`; `Cosmos DB`; `Blob Storage`; `Event Bus`; `MCP`; `OAuth 2.0`; `OIDC`; `Azure AD / Entra ID`; agent-to-agent protocols; `OWASP LLM Top 10`; jailbreak resistance; adversarial prompt hardening. Source: [Veeam](https://job-boards.greenhouse.io/veeamsoftware/jobs/4876895101) |

## Pairings That Look Real

| Pairing | Where seen | Read |
| --- | --- | --- |
| `MCP` + governed data access + auth | Carrot US; Exactera US; Veeam US; Forter UK | Protocol adoption is increasingly coupled to RBAC, OAuth, audit logging, and explicit scoping. |
| `Python` + `TypeScript` + SDK/platform responsibility | ClickHouse US; Carrot US; Bain AU; Smartsheet UK | Full-stack AI roles increasingly own both runtime orchestration and developer-facing platform surfaces. |
| observability stack + agents | ClickHouse US; Future US; Addepar US | Strong new shape: agents are being built either on top of telemetry platforms or with telemetry-first production expectations. |
| enterprise workflow tooling + agent systems | Carrot US; DeepIntent US; Forter UK | AI engineers are being hired as workflow-transformers, not just model/application builders. |
| graph substrate + retrieval + auditability | Exactera US; Catapult AU | Graph-backed reasoning remains niche, but the reasons for using it are now more explicit: defensibility, replayability, expert judgment capture. |
| evals + CI/CD + regression discipline | Cadence US; ClickHouse US; Awin UK; Future US | Evals are moving closer to software delivery discipline, not separate research work. |
| distributed training stack + efficiency benchmarking | Firmus AU | New in-region evidence that at least some AU AI engineering demand is deep infra/training oriented, not only app-layer agent work. |

## Language / Runtime Signal

| Runtime | Strength in sample | Typical shape |
| --- | --- | --- |
| `Python` | dominant | orchestration, evals, retrieval, automation, backend AI services |
| `TypeScript` / `Node.js` | strong second | AI product engineering, SDKs, MCP servers, internal tooling, platform interfaces |
| `SQL` | common | telemetry interrogation, governed retrieval, enterprise data access |
| `Go` / `Golang` | real secondary signal | agent platform infra, high-throughput systems, backend platform teams |
| `Java` / `C++` | selective | enterprise / systems-heavy forward-deployed roles |
| `Rust` | niche | performance-sensitive integrations and systems layers |
| `JAX` | niche but notable | distributed training roles only, not broad application engineering |

### Runtime read

- `Python` + `TypeScript` remains the most important pairing.
- `SQL` is more central than some AI narratives imply; it repeatedly shows up where AI systems need real governed data access.
- `Go` keeps appearing where the role owns high-throughput infra or hardened platform systems.

## Safety / Robustness Signal

### Common

- guardrails
- evaluation frameworks
- regression suites
- structured outputs
- observability
- human-in-the-loop
- reliability / operational ownership

### More specific this week

- HIPAA
- `SOC 2`
- RBAC
- `OAuth2`
- audit logging
- data residency requirements
- deterministic / auditable outcomes
- replayability via versioned prompts / models / indexes / graph state
- `OWASP LLM Top 10`
- jailbreak resistance
- adversarial prompt hardening

## Seen / Not Seen

### Seen clearly

- `MCP`
- `LangGraph`
- `Anthropic`
- `OpenAI`
- `Python`
- `TypeScript`
- `RAG`
- evaluation frameworks
- tracing / observability
- vector stores
- structured outputs

### Seen, but not dominant

- `LangChain`
- `LlamaIndex`
- `OpenTelemetry`
- `Databricks`
- `MLflow`
- `Pinecone`
- `Weaviate`
- `GraphRAG`
- `AutoGen`
- `Promptfoo`
- `AWS Bedrock`
- `Azure OpenAI Service`

### Weak or absent in this sample

- `Braintrust`
- `DeepEval`
- `TensorZero`
- `Helicone`
- `Milvus`
- `Qdrant`
- `FAISS`
- `Semantic Kernel`
- `Vercel AI SDK`
- `Mastra`
- `PydanticAI`

## Regional Read

| Region | Strongest pattern | Distinctive signal this week |
| --- | --- | --- |
| Australia | split between full-stack agent product roles and deeper infra/training roles | `TorchTitan`/`Megatron`/`FSDP` at Firmus and `LangGraph`/`AutoGen` in AI Ops at Caterpillar |
| United Kingdom | forward-deployed and internal-platform roles with strong protocol and adoption language | `MCP` servers, `Strands Agents`, `Pinecone`/`Weaviate`, and customer-embedded delivery remain prominent |
| United States | sharpest stack specificity and clearest platformization | `ClickHouse` agent layer, `Databricks` + `Unity Catalog` + `MLflow` + `MCP gateways`, `Workato` + `Claude skills`, and regulated-eval stacks |

## Source Set

- [Firmus - AI Engineer, AI & Applications](https://job-boards.greenhouse.io/firmus/jobs/5071736008)
- [Caterpillar - Agentic AI / AI Ops Engineer](https://careers.caterpillar.com/en/jobs/r0000376964/agentic-ai-ai-ops-engineer-platform-engineering/)
- [Bain - Full Stack AI Product Engineer (DataEdge)](https://www.bain.com/careers/find-a-role/position/?jobid=106304)
- [Karbon - Associate AI Engineer](https://job-boards.greenhouse.io/karbon/jobs/5995013004)
- [Catapult Sports - Principal AI Engineer](https://job-boards.greenhouse.io/catapultsports/jobs/7902420)
- [Smartsheet - Sr. Forward Deployed AI Engineer](https://job-boards.greenhouse.io/smartsheet/jobs/7930145)
- [Awin - Senior AI Engineer](https://job-boards.greenhouse.io/awin/jobs/7720582003)
- [Forter - Senior Software AI Engineer](https://job-boards.greenhouse.io/forter/jobs/8415897002)
- [Forter - Senior Applied AI Engineer](https://job-boards.greenhouse.io/forter/jobs/8486828002)
- [Palantir - Forward Deployed AI Engineer](https://jobs.lever.co/palantir/ff1029bd-bb6d-4d78-a03e-5f9744d0b798)
- [IFS - Forward Deployed AI Engineer](https://jobs.smartrecruiters.com/IFS1/744000128816717-forward-deployed-ai-engineer)
- [ClickHouse - AI Product Engineer - ClickStack](https://job-boards.greenhouse.io/clickhouse/jobs/6014112004)
- [Exactera - Principal AI Engineer](https://job-boards.greenhouse.io/exactera/jobs/7740360003)
- [Carrot - Applied AI Engineer, Enterprise](https://job-boards.greenhouse.io/carrotfertility/jobs/6011067004)
- [Cadence - Applied AI Engineer](https://www.cadence.care/jobs/applied-ai-engineer/)
- [DeepIntent - Applied AI Engineer](https://job-boards.greenhouse.io/deepintent/jobs/5979034004)
- [Addepar - Sr. Software Engineer - AI/ML - AI Platform](https://job-boards.greenhouse.io/addepar1/jobs/8479770002)
- [Capstone Investment Advisors - AI Engineer](https://job-boards.greenhouse.io/capstoneinvestmentadvisors/jobs/8453280002)
- [Future - Applied AI Engineer](https://job-boards.greenhouse.io/future/jobs/4683133005)
- [Veeam - Staff AI Engineer](https://job-boards.greenhouse.io/veeamsoftware/jobs/4876895101)
