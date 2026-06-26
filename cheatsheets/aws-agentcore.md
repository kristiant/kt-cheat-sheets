# Amazon Bedrock AgentCore

**What it is:** A set of managed services for running AI agents in production — serverless runtime, tool gateway, memory, identity, and observability. Framework-agnostic (works with LangGraph, Strands, CrewAI, raw code).

**Why people use it:** It's the *infrastructure* layer under an agent — session isolation, long-running execution, auth, and memory — so you don't build hosting, tool plumbing, and identity yourself. Bring any agent framework; AgentCore runs and scales it.

**Typically used for:** Deploying agents to production, exposing existing APIs/Lambdas/MCP servers as agent tools, giving agents persistent memory, and securing agent↔tool↔user access.

> GA October 2025. Build/deploy is **Python-first** (the SDK + `agentcore` CLI). Invoking a deployed agent works from any AWS SDK, including TypeScript (see below).

---

## The five components

| Component | What it does |
|---|---|
| **Runtime** | Serverless, session-isolated host for agents. Up to 8-hour executions; streaming; A2A protocol. |
| **Gateway** | Turns APIs, Lambda functions, and MCP servers into agent tools (IAM + OAuth auth). |
| **Memory** | Managed short- and long-term memory — store and recall context across sessions. |
| **Identity** | Agent identity + delegated auth (OAuth, Cognito/Okta/Entra) so agents act as a user or themselves. |
| **Observability** | Traces, metrics, and logs for agent runs (CloudWatch / OpenTelemetry). |

Mix and match — Runtime alone hosts an agent; add Gateway/Memory/Identity as needed.

---

## Build & deploy an agent (Python)

The Runtime SDK wraps your agent as an HTTP service (`/invocations` + `/ping`) — you write the logic, it handles the server.

```bash
pip install bedrock-agentcore bedrock-agentcore-starter-toolkit
```

```python
# agent.py — framework-agnostic; here with Strands, but LangGraph etc. work the same
from bedrock_agentcore.runtime import BedrockAgentCoreApp
from strands import Agent

app = BedrockAgentCoreApp()
agent = Agent()

@app.entrypoint
def invoke(payload):
    result = agent(payload.get("prompt", "Hello"))
    return {"result": result.message}

if __name__ == "__main__":
    app.run()
```

Deploy with the CLI — it containerises, pushes to ECR, and creates the runtime:

```bash
agentcore configure --entrypoint agent.py
agentcore launch -l          # run locally for testing
agentcore launch             # build → ECR → deploy to AgentCore Runtime
agentcore invoke '{"prompt": "hello"}'
```

---

## Invoke a deployed agent (TypeScript)

Once deployed, call the runtime by ARN from any app. Streaming response, session-aware.

```bash
npm i @aws-sdk/client-bedrock-agentcore
```

```ts
import { BedrockAgentCoreClient, InvokeAgentRuntimeCommand }
  from "@aws-sdk/client-bedrock-agentcore";

const client = new BedrockAgentCoreClient({ region: "us-east-1" });

const res = await client.send(new InvokeAgentRuntimeCommand({
  agentRuntimeArn: process.env.AGENT_ARN,
  runtimeSessionId: "user-42",                       // maintains conversation context
  payload: new TextEncoder().encode(JSON.stringify({ prompt: "hello" })),
}));

// response streams back as chunks
const text = await res.response?.transformToString();
```

> So you can host the agent in AgentCore (Python) but drive it from a TS backend ([aws-lambda.md](aws-lambda.md), [nodejs.md](nodejs.md)).

---

## Advanced

### Gateway — existing services as agent tools

Point Gateway at a Lambda, an OpenAPI spec, or an MCP server; it exposes them as tools the agent can call, handling auth. No glue code per tool.

```
Lambda / OpenAPI / MCP server  ──►  AgentCore Gateway  ──►  agent tools (IAM or OAuth)
```

> Gateway speaks **MCP**, so an agent reaches Gateway tools the same way it reaches any MCP server. See [agentic-patterns.md](../practices/agentic-patterns.md).

### Memory — persistent context

```python
from bedrock_agentcore.memory import MemoryClient

memory = MemoryClient()
memory.create_event(memory_id, actor_id="user-42", payload=[...])   # store
recent = memory.list_events(memory_id, actor_id="user-42")          # recall short-term
```

Long-term strategies (summaries, user preferences, semantic facts) are extracted and consolidated for you, or self-managed for full control.

### Identity — agents acting on behalf of users

AgentCore Identity issues each agent an identity and brokers OAuth tokens (stored in a secure vault), so an agent can call a downstream service *as the user* with scoped access — not with a shared static key.

### Built-in tools

- **Code Interpreter** — sandboxed code execution for data/maths tasks.
- **Browser** — managed headless browser for web tasks.

Both are managed, isolated, and callable from the agent without you running the sandbox.

### Observability

Enable tracing to get per-step spans (model calls, tool calls, memory ops) in CloudWatch / any OTEL backend — essential for debugging multi-step agent runs.

---

## Practical recipes

**Local dev loop (no deploy):**
```bash
agentcore launch -l && agentcore invoke '{"prompt":"test"}'
```

**Deploy and grab the runtime ARN:**
```bash
agentcore launch
agentcore status            # shows the AgentRuntime ARN to invoke later
```

**Invoke a deployed agent from the CLI:**
```bash
aws bedrock-agentcore invoke-agent-runtime \
  --agent-runtime-arn "$ARN" --runtime-session-id user-42 \
  --payload '{"prompt":"hello"}' out.json
```

**Minimal IAM to invoke:**
```json
{ "Effect": "Allow", "Action": "bedrock-agentcore:InvokeAgentRuntime", "Resource": "<runtime-arn>" }
```

---

## Tips

- AgentCore is the **infra layer**, not an agent framework — bring LangGraph/Strands/etc.; it hosts them.
- Use components à la carte: Runtime to host, Gateway for tools, Memory for context, Identity for delegated auth.
- Build/deploy is Python (`agentcore` CLI + SDK); invoke from any SDK, so a TS app can front a Python-hosted agent.
- Prefer **Gateway** over hand-wiring tool auth — it turns Lambdas/APIs/MCP servers into tools with IAM/OAuth handled.
- Use **Identity** delegation instead of giving an agent a static downstream key.
- Turn on Observability early — multi-step agent runs are very hard to debug blind.
- The newer `agentcore` CLI supersedes the legacy starter toolkit for new projects.
