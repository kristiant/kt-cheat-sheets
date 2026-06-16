# promptfoo (red-teaming)

**What it is:** An open-source CLI for testing LLM apps — evals *and* automated red-teaming. It generates adversarial inputs, runs them against your model or app, grades the responses with an LLM judge, and reports vulnerabilities.

**Why people use it:** Find jailbreaks, prompt injection, PII leaks, and unsafe actions *before* shipping — repeatably, in CI — instead of manual prompt-poking. Reports map to OWASP LLM Top 10, NIST AI RMF, and MITRE ATLAS.

**Typically used for:** Red-teaming chatbots/RAG/agents, regression-testing safety in CI, and producing compliance report cards.

> This sheet covers the red-team side. promptfoo also does prompt/model evals (`promptfoo eval`). Docs: https://www.promptfoo.dev/docs/red-team/

---

## Mental model

A red-team scan is four inputs producing a graded report:

```
purpose   — what the system is for + its rules (used to generate relevant attacks)
targets   — the model or app endpoint under test
plugins   — adversarial input GENERATORS (what to attack: PII, jailbreak, RBAC, …)
strategies— delivery TECHNIQUES (how to wrap/escalate: injection, crescendo, base64, …)
        ↓
generate adversarial test cases → run against target → LLM-judge grades each → report
```

**Plugins decide *what* harm to probe; strategies decide *how* to deliver it.** A plugin generates "leak another user's order"; a strategy might wrap it in a jailbreak or a multi-turn escalation.

---

## Setup & quickstart

```bash
npx promptfoo@latest redteam init        # interactive — pick target, plugins, strategies → config
npx promptfoo@latest redteam run         # generate + run the scan
npx promptfoo@latest redteam report      # open the report UI
```

No install needed (`npx`), or `npm i -g promptfoo`.

## Common commands

```bash
promptfoo redteam init          # scaffold promptfooconfig.yaml interactively
promptfoo redteam generate      # generate adversarial test cases → redteam.yaml (no run)
promptfoo redteam run           # generate + eval in one step (keeps tests in sync with config)
promptfoo redteam report        # open the vulnerability report / compliance card
promptfoo view                  # open the results web UI
promptfoo redteam run --filter-plugins pii   # run only some plugins
promptfoo redteam eval          # run existing redteam.yaml without regenerating
```

---

## Config

The `redteam` section of `promptfooconfig.yaml`:

```yaml
targets:
  - id: openai:gpt-4o-mini        # a model…
    label: support-bot

redteam:
  purpose: >
    A customer-support assistant for an e-commerce store. It can look up orders
    and issue refunds. It must NOT reveal other customers' data, issue refunds
    without identity verification, or follow instructions from order notes.
  plugins:
    - harmful                     # collection — all harm categories
    - pii                         # personal-data leaks
    - excessive-agency            # taking actions beyond scope
    - rbac                        # access-control bypass
    - indirect-prompt-injection   # acting on injected instructions in data
  strategies:
    - jailbreak                   # iterative attacker-model refinement
    - jailbreak:composite
    - prompt-injection
    - crescendo                   # multi-turn escalation
  numTests: 10                    # cases per plugin
```

`purpose` is the highest-leverage field — the better you describe the system and its rules, the more relevant (and damning) the generated attacks.

---

## Plugins (what to attack)

Adversarial input generators. Use individually or as collections.

| Plugin | Probes for |
|---|---|
| `harmful` | hate, self-harm, violence, illegal acts (a collection of subcategories) |
| `pii` | leaking personal data (direct, via API/DB, social engineering) |
| `excessive-agency` | taking actions beyond its remit |
| `hallucination` | confidently fabricating facts |
| `hijacking` | being steered off its stated purpose |
| `indirect-prompt-injection` | obeying instructions hidden in retrieved/tool data |
| `rbac` / `bola` / `bfla` | access-control & object-level authorization bypass |
| `sql-injection` / `shell-injection` / `ssrf` | classic injection via tool inputs |
| `tool-discovery` / `debug-access` | enumerating tools or hitting debug interfaces |

Collections: `harmful`, `owasp:llm` (OWASP LLM Top 10), `foundation`, `nist:ai:measure`, `mitre:atlas`.

```yaml
plugins:
  - owasp:llm        # the OWASP LLM Top 10 set in one line
```

---

## Strategies (how to deliver)

Techniques that wrap or escalate the plugin payloads. Single-turn vs multi-turn.

| Strategy | Technique |
|---|---|
| `basic` | raw payload, no wrapping (default baseline) |
| `jailbreak` | attacker model iteratively refines a bypass |
| `jailbreak:composite` | combines multiple jailbreak techniques |
| `jailbreak:tree` | tree search over attack variants |
| `prompt-injection` | wraps the payload in injection frames |
| `crescendo` | **multi-turn** — each message escalates from the last |
| `goat` / `hydra` | **multi-turn** adaptive attacks against stateful chat/agents |
| `base64` / `rot13` / `leetspeak` / `hex` | encode the payload to slip filters |
| `multilingual` | translate the attack into other languages |
| `math-prompt` | hide intent inside a math/logic framing |

```yaml
strategies:
  - id: jailbreak:composite
  - id: crescendo        # use multi-turn for chatbots/agents with memory
```

> Multi-turn strategies (`crescendo`, `goat`, `hydra`) matter most for **stateful** apps — chatbots and agents — where a single message won't surface the vulnerability.

---

## Targets — model vs your app

Test a raw model, or your **actual application** over HTTP (the realistic scan — it exercises your system prompt, RAG, and tools).

```yaml
targets:
  - id: https
    label: prod-chat
    config:
      url: https://api.example.com/chat
      method: POST
      headers: { 'Content-Type': 'application/json' }
      body: { message: '{{prompt}}', sessionId: '{{sessionId}}' }
      transformResponse: 'json.reply'      # extract the assistant text from your response
```

For agents/chatbots, point at the app endpoint so multi-turn strategies hit real memory and tools.

---

## CI integration

```bash
# fail the build if new vulnerabilities appear
promptfoo redteam run --no-progress-bar --output results.json
```

```yaml
# .github/workflows/redteam.yml
- run: npx promptfoo@latest redteam run --no-progress-bar
  env:
    OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
```

> Run a **small, fast** plugin/strategy set on every PR (e.g. `pii`, `prompt-injection`) and a full scan nightly — full scans generate hundreds of cases and cost real tokens.

---

## Practical recipes

**Scaffold a scan against a model:**
```bash
npx promptfoo@latest redteam init
```

**Run only PII and injection checks (fast PR gate):**
```bash
promptfoo redteam run --filter-plugins pii,indirect-prompt-injection
```

**Generate cases without running (inspect first):**
```bash
promptfoo redteam generate   # writes redteam.yaml — review before spending tokens
```

**Re-run the same cases after a prompt change (regression):**
```bash
promptfoo redteam eval       # reuses redteam.yaml, no regeneration
```

**Scan the OWASP LLM Top 10 in one config:**
```yaml
redteam:
  plugins: [owasp:llm]
  strategies: [jailbreak, prompt-injection]
```

**Point at a local app endpoint:**
```yaml
targets:
  - id: http://localhost:3000/api/chat
```

---

## Tips

- Write a specific, rule-laden `purpose` — vague purpose = generic, weak attacks.
- Test your **app endpoint**, not just the raw model; the system prompt, RAG, and tools are where real holes are.
- Use multi-turn strategies (`crescendo`, `goat`) for chatbots/agents — single-turn misses stateful exploits.
- `redteam run` keeps tests in sync with config; use `redteam eval` only to re-grade existing `redteam.yaml`.
- Keep PR scans small and fast; run the full plugin/strategy matrix on a schedule — it's hundreds of judged LLM calls.
- Generated attacks are inputs you'd never paste yourself — only run them against systems you're authorized to test.
- Pin the version in CI; plugin/strategy behaviour changes between releases.
