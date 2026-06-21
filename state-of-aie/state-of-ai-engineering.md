# State of AI Engineering

Date: 2026-06-22
Scope: sampled current AI Engineer / Applied AI Engineer / AI Product Engineer / ML Engineer postings across Australia, the US, and the UK.
Method: treat postings as telemetry; record concrete named technical signals visible in job pages on 2026-06-22. Sample favors official company job pages over aggregators.

## Week-over-Week Change

### New or materially clearer this week

| Signal | Evidence | Read |
| --- | --- | --- |
| `AWS Strands` / `Strands SDK` | Hibu US; Ripple US; Smartsheet UK (nice-to-have OSS contributions); Komodo US | The AWS-native agent stack is no longer background noise. It is now named alongside `MCP`, `LangGraph`, and eval infra. |
| `AgentCore` | Hibu US; Ripple US | New concrete runtime/orchestration signal. Last week had workflow durability adjacency; this week has named AWS agent runtime components. |
| `A2A` / agent-to-agent protocols | Veeam US; Medeloop US | Protocol language is getting more explicit. Teams are no longer only saying "multi-agent"; they are naming communication patterns. |
| `Deep Agents` | Awin UK | New framework signal in a production product-engineering role, paired with `LangGraph` rather than replacing it. |
| `OWASP LLM Top 10` | Veeam US | Security language is becoming more operational and standards-shaped, not just generic "guardrails". |
| `web browsing` / `computer use` as platform capabilities | Addepar US | Cross-surface agent capability is now showing up in platform job requirements, not only demos or research chatter. |
| `Claude Code` / agent harness language | Ripple US; Anthropic UK public-sector role (`Claude Code` support) | Developer-agent tooling moved closer to hiring-language center. |
| `msgspec` | Snorkel AI US | Typed Python serialization is a small but high-signal implementation detail that did not show up clearly last week. |

### Stronger than last week

- `MCP`
  - Still the strongest protocol signal in the sample, but now described as internal MCP servers, MCP resource packs, MCP server/client builds, and secure tool exposure contracts.
- AWS-first agent infrastructure
  - `AWS Bedrock` is still common, but now paired with `Strands`, `AgentCore`, `EKS`, `Istio`, and deployment patterns rather than just model access.
- Security and compliance around agent systems
  - `RBAC`, `OAuth 2.0`, `OIDC`, `query constraints`, `safe tool boundaries`, `GDPR`, `EU AI Act`, `DORA`, `NIS2`, `Zero Data Retention`, and `OWASP LLM Top 10` all surfaced in-region this week.
- TypeScript as a real AI runtime language
  - The `Python` + `TypeScript` pairing remains stronger than either alone, with product-facing orchestration, SDKs, agent UIs, and typed contracts now explicit again.

### Weaker, thinner, or newly absent in this sample

- `PydanticAI`
  - Clear last week; not a strong in-region signal this week.
- `Mastra`
  - Visible last week; absent from this sampled AU/US/UK set.
- `Braintrust`
  - Still weaker than `LangSmith`, `Langfuse`, and generic observability + eval stacks.
- `Promptfoo`
  - Still real, but again concentrated in one standout AU role rather than broadly recurring.
- `Google ADK`
  - Still present, but only as a secondary AU signal rather than a recurring cross-region stack choice this week.

### Trend call

The direction that looks more real now than it did last week: teams are converging on agent runtime operations as a first-class engineering domain. The postings care less about "which framework wins" and more about protocol boundaries, deployment kits, runtime observability, secure tool exposure, eval pipelines, and cross-team platform reuse.

## Signal Map

### Strongest signals in the sample

| Category | Stronger signals | Notes |
| --- | --- | --- |
| Agent frameworks / protocols | `MCP`, `LangGraph` | Still the center of gravity. |
| Cloud / model platforms | `AWS Bedrock`, `Anthropic`, `OpenAI`, `Azure OpenAI Service` | `AWS` remains the most explicit cloud anchor. |
| Eval / observability | `LangSmith`, `Langfuse`, `OpenTelemetry`, regression tests, automated eval pipelines | Vendor mix is widening, but many ads still describe capability more than a single tool. |
| Languages / runtimes | `Python`, `TypeScript`, `Node.js`, `SQL` | Bilingual product/runtime roles remain common. |
| Safety / robustness | guardrails, hallucination reduction, prompt injection defenses, HITL, auditability, policy enforcement | More concrete than last week. |

### Sharper niche signals

| Category | Signals |
| --- | --- |
| AWS-native agent stack | `Strands`, `AgentCore`, `AWS Bedrock`, `EKS`, `Istio` |
| Security / identity | `OAuth 2.0`, `OIDC`, `Azure AD / Entra ID`, `RBAC`, `Zero Data Retention`, `OWASP LLM Top 10` |
| Retrieval / memory variants | `Neo4j`, `GraphRAG`, vector DBs, hybrid retrieval, memory/state systems |
| Platform capabilities | web browsing, computer use, agent SDKs, deployment kits, resource packs |
| Inference / serving | `vLLM` |

## Company -> Tool Matrix

### Australia

| Company | Role | Concrete signal |
| --- | --- | --- |
| Anthropic | Applied AI Engineer | `Claude`; evaluation frameworks; Python; customer-facing architecture design for LLM products. Source: [Anthropic](https://job-boards.greenhouse.io/anthropic/jobs/5248983008) |
| Bain (`DataEdge`) | Full Stack AI Product Engineer | `TypeScript`; `Node.js`; `Python`; `LangGraph`; `FastAPI`; `Pydantic v2`; `pytest`; `Temporal`; `pgvector`; `OpenSearch`; `Docker`; `Kubernetes`; `tRPC`; `Zod`; `OpenAPI codegen`; `Vitest`; `Jest`; `Vite`; `Turbopack`; `esbuild`. Source: [Bain](https://www.bain.com/careers/find-a-role/position/?jobid=106304) |
| Karbon | Associate AI Engineer | `RAG`; chaining; `MCP`; `Python`; `sklearn`; `PyTorch`; `TensorFlow`; `spaCy`; `ADK`; `LangGraph`; `Agent SDK`; `Snowflake`; `dbt`; `C#`; `Azure`. Source: [Karbon](https://job-boards.greenhouse.io/karbon/jobs/5995013004) |
| Catapult Sports | Principal AI Engineer | `Go`; `Rust`; `Python`; `React`; `Neo4j`; `GraphRAG`; `AWS Bedrock`; `Promptfoo`; map-reduce over graph communities; LLM-as-judge; custom eval frameworks. Source: [Catapult Sports](https://job-boards.greenhouse.io/catapultsports/jobs/7902420) |

### United Kingdom

| Company | Role | Concrete signal |
| --- | --- | --- |
| Smartsheet | Sr. Forward Deployed AI Engineer | multi-agent orchestration; `MCP` resource packs; `Python`; `TypeScript`/`JavaScript`; deployment kits; agent deployment at scale; nice-to-have `Strands SDK`, `LangChain`, open-source MCP servers, `AWS Bedrock`, `Temporal`, `Databricks`. Source: [Smartsheet](https://job-boards.greenhouse.io/smartsheet/jobs/7930145) |
| Awin | Senior AI Engineer | `AWS`; `LangGraph`; `Deep Agents`; `LangSmith`; `MCP`; vector databases; hybrid retrieval; structured evals; regression tests; tracing/observability; caching and routing strategies. Source: [Awin](https://job-boards.greenhouse.io/awin/jobs/7720582003) |
| Forter | Senior Software AI Engineer | internal `MCP` server; `RAG`; evaluation frameworks; AI guardrails; secure exposure of internal data stores; `Strands Agents` in related role requirements. Source: [Forter](https://job-boards.greenhouse.io/forter/jobs/8415897002) |
| Capco | Lead AI Engineer | multi-modal LLMs; `LangChain`; `LlamaIndex`; `Langfuse`; `LangSmith`; `MCP`; bias/hallucination mitigation; Python; cloud-native AI deployment. Source: [Capco](https://job-boards.greenhouse.io/capco/jobs/6870486) |
| Anthropic | Applied AI Security Architect | `MCP`; `RBAC`; `GDPR`; `EU AI Act`; `DORA`; `NIS2`; `Zero Data Retention`; `AWS`; `Azure`; `GCP`; audit logging; private endpoints; API security. Source: [Anthropic](https://job-boards.greenhouse.io/anthropic/jobs/5227641008) |
| Anthropic | Applied AI Architect, Public Sector | `Claude Code`; `Claude API`; `Claude for Enterprise`; evaluation frameworks; UK public-sector security/governance standards. Source: [Anthropic](https://job-boards.greenhouse.io/anthropic/jobs/5227643008) |

### United States

| Company | Role | Concrete signal |
| --- | --- | --- |
| Hibu | Applied AI Engineer | `AWS Bedrock`; `MCP`; `Salesforce`; `Claude`/`GPT`; `RAG`; webhooks; `Vertex AI`; `Azure AI`; `AWS Strands Agents`; `Claude Managed Agents`; `AWS AgentCore`. Source: [Hibu](https://job-boards.greenhouse.io/hibu/jobs/4696330005) |
| Ripple | Staff Software Engineer, GenAI Platform | `Python`; `Go`; `Java`; `LangGraph`; `MCP`; `Claude Code`; `AgentCore`; `Strands`; `LangChain`; `EKS`; `Istio`; `OpenTelemetry`; `Prometheus`; `Grafana`; `Qdrant`; `Tree-sitter`. Source: [Ripple](https://job-boards.greenhouse.io/ripple/jobs/7918366) |
| Snorkel AI | Applied AI Engineer, Federal | `pydantic`; `mypy`; `pytest`; `poetry`; `FastAPI`; `msgspec`; `Ray`; `Airflow`; `scikit-learn`; `PyTorch`; `Hugging Face Transformers`; `FAISS`; `Spark`; `Chroma`; `Weaviate`; `LlamaIndex`; `LangGraph`; `CrewAI`. Source: [Snorkel AI](https://job-boards.greenhouse.io/snorkelai/jobs/5721276004) |
| Addepar | Sr. Software Engineer, AI Platform | `Python`; `Golang`; `gRPC`; `AWS`; `Kubernetes`; MCP / cross-platform tool use; web browsing; computer use; `Databricks`; `LangChain`; `MLflow`. Source: [Addepar](https://job-boards.greenhouse.io/addepar1/jobs/8479770002) |
| Future | Applied AI Engineer | `LangChain`; `LangGraph`; tool/function calling; structured output; `Pydantic`; `JSON Schema`; `SSE`; webhooks; `AWS Bedrock`; `ECR`; `CloudFront`; `S3`; `Cognito`; `Langfuse`; `OpenTelemetry`; `Datadog`. Source: [Future](https://job-boards.greenhouse.io/future/jobs/4683133005) |
| Komodo Health | Full-Stack AI Engineer | `vLLM`; `CrewAI`; `Strands`; `OpenAI` / Chat Completions APIs; `Spark`; `Snowflake`; `Databricks`; monitoring, evaluation, and governance across the "full intelligence stack". Source: [Komodo Health](https://job-boards.greenhouse.io/komodohealth/jobs/8584991002) |
| Medeloop | Staff AI Machine Learning Engineer | `MCP`; `A2A`; `LangChain`; `LangGraph`; `Hugging Face`; `PyTorch`; vector databases; `LangSmith`; `Phoenix`; `ReAct`; `CoT`; `ToT`; adversarial testing; HITL. Source: [Medeloop](https://job-boards.greenhouse.io/medeloop/jobs/4236722009) |
| Veeam | Staff AI Engineer | `Azure OpenAI Service`; `Kubernetes`; `Cosmos DB`; `Blob Storage`; `Event Bus`; `MCP`; `OAuth 2.0`; `OIDC`; `Azure AD / Entra ID`; agent-to-agent protocols; OWASP LLM Top 10; jailbreak resistance; adversarial prompt hardening. Source: [Veeam](https://job-boards.greenhouse.io/veeamsoftware/jobs/4876895101) |
| Capstone Investment Advisors | AI Engineer | `Python`; `LangChain`; `LlamaIndex`; vector stores; `OpenAI API`; `Anthropic API`; MCP servers/clients; RAG; function calling. Source: [Capstone](https://job-boards.greenhouse.io/capstoneinvestmentadvisors/jobs/8453280002) |

## Pairings That Look Real

| Pairing | Where seen | Why it matters |
| --- | --- | --- |
| `AWS Bedrock` + `Strands` + `AgentCore` + `MCP` | Hibu US; Ripple US (runtime overlap) | Strongest new stack-combination this week. AWS-native agent runtime language is becoming explicit. |
| `LangGraph` + `MCP` | Bain AU; Karbon AU; Awin UK; Ripple US | Still the most stable orchestration + protocol pairing. |
| `MCP` + identity / policy controls | Veeam US; Anthropic UK | Protocol work is now coupled to `RBAC`, `OAuth`, `OIDC`, auditability, and secure tool boundaries. |
| `TypeScript` + `Python` + durable workflow thinking | Bain AU; Smartsheet UK | Full-stack AI product roles are explicitly spanning typed UI/services and backend agent systems. |
| `Neo4j` + `GraphRAG` + `Promptfoo` + `AWS Bedrock` | Catapult AU | Graph-native retrieval remains niche but high-signal and coherent when it appears. |
| `OpenTelemetry` + AI-specific eval / traces | Ripple US; Future US | AI telemetry is converging with standard platform observability. |
| `LangSmith` + eval-driven product engineering | Awin UK; Capco UK | `LangSmith` remains a strong named anchor when teams discuss tracing and regression. |
| `Claude Code` + agent platform work | Ripple US; Anthropic UK public sector | Developer-agent surfaces are entering the hiring baseline. |

## Language / Runtime Signal

| Runtime | Strength in sample | Typical role shape |
| --- | --- | --- |
| `Python` | dominant | orchestration, evals, backend AI services, platform layers |
| `TypeScript` / `Node.js` | strong second | AI product surfaces, SDKs, full-stack agent systems, deployment tooling |
| `SQL` | common | retrieval, analytics, data products, warehouse-backed AI |
| `Go` | sharper than last week | systems-heavy agent services and platform infra |
| `Java` / `Golang` / `gRPC` | supporting but real | platform and enterprise integration roles |
| `Rust` | still niche | performance-sensitive paths, not general baseline |

### Runtime read

- The strongest hiring shape is still `Python` + `TypeScript`.
- `TypeScript` is now repeatedly tied to agent deployment kits, SDKs, orchestration surfaces, and trustworthy UI patterns, not just frontend work.
- `Go` remains a high-signal secondary runtime when the role owns core agent services or performance-sensitive infra.

## Safety / Robustness Signal

### Common

- guardrails
- evaluation frameworks
- regression testing
- human-in-the-loop
- hallucination reduction
- structured outputs
- auditability

### More specific this week

- `OWASP LLM Top 10`
- adversarial prompt hardening
- jailbreak resistance
- `Zero Data Retention`
- `RBAC` / auth-context propagation
- query constraints and safe tool boundaries
- `GDPR`, `EU AI Act`, `DORA`, `NIS2`
- bias / hallucination mitigation as named role language

## Seen / Not Seen

### Seen clearly

- `MCP`
- `LangGraph`
- `AWS Bedrock`
- `Anthropic`
- `OpenAI`
- `Python`
- `TypeScript`
- `LangSmith`
- `Langfuse`
- `OpenTelemetry`
- `vLLM`
- `Strands`
- `AgentCore`

### Seen, but not dominant

- `LlamaIndex`
- `CrewAI`
- `Azure OpenAI Service`
- `Databricks`
- `MLflow`
- `Neo4j`
- `GraphRAG`
- `Promptfoo`
- `Deep Agents`
- `Claude Code`

### Weak or absent in this sample

- `Mastra`
- `PydanticAI`
- `Braintrust`
- `TensorZero`
- `Helicone`
- `Milvus`
- `turbopuffer`
- `Semantic Kernel`
- `Vercel AI SDK`

## Regional Read

| Region | Strongest pattern | Distinctive signal this week |
| --- | --- | --- |
| Australia | full-stack product AI plus graph-heavy experimentation | `TypeScript`-first product engineering at Bain and graph-native eval-heavy architecture at Catapult |
| United Kingdom | deployment/integration roles with stronger governance language | `MCP` resource packs, `Deep Agents`, and explicit UK/EU regulatory framing |
| United States | platform-heavy agent engineering with sharper runtime specificity | `Strands`, `AgentCore`, `OpenTelemetry`, `Qdrant`, `Tree-sitter`, `Azure OpenAI Service`, and explicit A2A/security boundary work |

## Source Set

- [Anthropic - Applied AI Engineer (Sydney)](https://job-boards.greenhouse.io/anthropic/jobs/5248983008)
- [Bain - Full Stack AI Product Engineer (DataEdge)](https://www.bain.com/careers/find-a-role/position/?jobid=106304)
- [Karbon - Associate AI Engineer](https://job-boards.greenhouse.io/karbon/jobs/5995013004)
- [Catapult Sports - Principal AI Engineer](https://job-boards.greenhouse.io/catapultsports/jobs/7902420)
- [Smartsheet - Sr. Forward Deployed AI Engineer (UK)](https://job-boards.greenhouse.io/smartsheet/jobs/7930145)
- [Awin - Senior AI Engineer](https://job-boards.greenhouse.io/awin/jobs/7720582003)
- [Forter - Senior Software AI Engineer](https://job-boards.greenhouse.io/forter/jobs/8415897002)
- [Capco - Lead AI Engineer](https://job-boards.greenhouse.io/capco/jobs/6870486)
- [Anthropic - Applied AI Security Architect](https://job-boards.greenhouse.io/anthropic/jobs/5227641008)
- [Anthropic - Applied AI Architect, Public Sector](https://job-boards.greenhouse.io/anthropic/jobs/5227643008)
- [Hibu - Applied AI Engineer](https://job-boards.greenhouse.io/hibu/jobs/4696330005)
- [Ripple - Staff Software Engineer, GenAI Platform](https://job-boards.greenhouse.io/ripple/jobs/7918366)
- [Snorkel AI - Applied AI Engineer, Federal](https://job-boards.greenhouse.io/snorkelai/jobs/5721276004)
- [Addepar - Sr. Software Engineer, AI Platform](https://job-boards.greenhouse.io/addepar1/jobs/8479770002)
- [Future - Applied AI Engineer](https://job-boards.greenhouse.io/future/jobs/4683133005)
- [Komodo Health - Full-Stack AI Engineer](https://job-boards.greenhouse.io/komodohealth/jobs/8584991002)
- [Medeloop - Staff AI Machine Learning Engineer](https://job-boards.greenhouse.io/medeloop/jobs/4236722009)
- [Veeam - Staff AI Engineer](https://job-boards.greenhouse.io/veeamsoftware/jobs/4876895101)
- [Capstone Investment Advisors - AI Engineer](https://job-boards.greenhouse.io/capstoneinvestmentadvisors/jobs/8453280002)
