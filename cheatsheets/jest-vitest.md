# Jest & Vitest

**What they are:** The two dominant JS/TS test frameworks. **Jest** (Meta) is the long-time default; **Vitest** is the Vite-native successor with a near-identical, Jest-compatible API.

**Why people use them:** Fast, structured tests with assertions, mocking, fixtures, snapshots, and coverage — the safety net for refactoring and CI gates.

**Typically used for:** Unit and integration tests, TDD, regression suites. In agent code, both mock the LLM so you test *your* logic (routing, tool selection, parsing) deterministically without API calls.

> The APIs are ~95% the same. Vitest is the default for new Vite/TS projects (ESM-native, faster); Jest still dominates existing React/Node codebases. **Mostly: swap `jest` → `vi`.**

---

## Jest ↔ Vitest

| | Jest | Vitest |
|---|---|---|
| Run | `jest` | `vitest` (watch) / `vitest run` |
| Mock fn | `jest.fn()` | `vi.fn()` |
| Mock module | `jest.mock()` | `vi.mock()` |
| Spy | `jest.spyOn()` | `vi.spyOn()` |
| Fake timers | `jest.useFakeTimers()` | `vi.useFakeTimers()` |
| Config | `jest.config.js` | `vitest.config.ts` |
| Globals | on by default | opt-in (`globals: true`) or import |

`describe` / `it` / `test` / `expect` / `beforeEach` are identical in both.

```bash
# Vitest
npm i -D vitest
npx vitest                # watch
npx vitest run            # single run (CI)
npx vitest run --coverage

# Jest
npm i -D jest
npx jest
npx jest --coverage
```

---

## Basics (same in both)

```ts
import { describe, it, expect, beforeEach } from "vitest";   // omit import in Jest, or with globals:true
import { parseIntent } from "../src/intent";

describe("parseIntent", () => {
  beforeEach(() => { /* fixtures */ });

  it("classifies a refund request", () => {
    expect(parseIntent("I want my money back")).toBe("refund");
  });

  it.each([
    ["cancel my plan", "cancel"],
    ["where's my order", "status"],
  ])("maps %s → %s", (input, expected) => {
    expect(parseIntent(input)).toBe(expected);
  });
});
```

### Common matchers

```ts
expect(x).toBe(3);                 // Object.is / primitives
expect(obj).toEqual({ a: 1 });     // deep equality
expect(obj).toMatchObject({ a: 1 }); // partial
expect(arr).toContain("x");
expect(fn).toThrow("boom");
expect(val).toBeNull();
expect(spy).toHaveBeenCalledWith(1, 2);
await expect(promise).resolves.toBe(42);
await expect(promise).rejects.toThrow();
```

---

## Mocking

> For typed whole-object mocks (`mock<T>()`), shared hoisted fixtures, AAA structure, `nock`, and mocking the LLM — i.e. how to *structure* real backend tests — see [practices/typescript-testing-patterns.md](../practices/typescript-testing-patterns.md).

```ts
import { vi, expect, it } from "vitest";   // jest in Jest

const fn = vi.fn().mockReturnValue(7);
fn(1, 2);
expect(fn).toHaveBeenCalledWith(1, 2);

// Mock an entire module
vi.mock("../src/db", () => ({ getUser: vi.fn().mockResolvedValue({ id: 1 }) }));

// Spy on a real method, restore after
const spy = vi.spyOn(console, "log").mockImplementation(() => {});

// Resolve/reject async
vi.fn().mockResolvedValue("ok");
vi.fn().mockRejectedValueOnce(new Error("fail"));   // once, then falls through
```

---

## Advanced — testing agents

### Mock the LLM, assert on your logic

Never hit a real model in unit tests. Inject a fake chat model and assert your graph/tool wiring behaves deterministically.

```ts
import { it, expect } from "vitest";
import { FakeListChatModel } from "@langchain/core/utils/testing";

it("routes a billing query to the billing tool", async () => {
  const llm = new FakeListChatModel({ responses: ["billing"] });   // canned, no network

  const route = await classify(llm, "my card was charged twice");
  expect(route).toBe("billing");
});
```

> **Grounded:** LangChain ships [`FakeListChatModel`](https://js.langchain.com/docs/integrations/chat/fake) / `FakeChatModel` in `@langchain/core/utils/testing` precisely so chains and graphs can be unit-tested without API calls.

### Assert a tool was called

```ts
const search = vi.fn().mockResolvedValue("result");
const agent = buildAgent({ tools: [makeTool("search", search)], llm });

await agent.invoke({ messages: [userMsg("find X")] });
expect(search).toHaveBeenCalledOnce();
```

### Fake timers (retry/backoff logic)

```ts
vi.useFakeTimers();
const p = retryWithBackoff(flakyFn);
await vi.runAllTimersAsync();        // fast-forward through delays — no real waiting
vi.useRealTimers();
```

### Snapshot a prompt template (catch drift)

```ts
expect(buildPrompt(context)).toMatchSnapshot();
```

---

## Practical recipes

**Run one file / one test by name:**
```bash
npx vitest run src/intent.test.ts
npx vitest run -t "classifies a refund"     # Jest: jest -t "classifies a refund"
```

**Coverage gate in CI (Vitest):**
```bash
npx vitest run --coverage --coverage.thresholds.lines=80
```

**Only run tests affected by changes:**
```bash
npx vitest --changed              # Jest: jest --onlyChanged
```

**Update snapshots after an intentional change:**
```bash
npx vitest run -u                 # Jest: jest -u
```

**Reset mocks between tests (config):**
```ts
// vitest.config.ts → test: { clearMocks: true, restoreMocks: true }
// jest.config.js  → { clearMocks: true, restoreMocks: true }
```

---

## Migrating Jest → Vitest

- Add `import { ... } from "vitest"` (or set `globals: true` in config to keep bare `describe`/`it`).
- Find-replace `jest.` → `vi.` (covers `fn`, `mock`, `spyOn`, `useFakeTimers`).
- Move config from `jest.config.js` to `vitest.config.ts` (under a `test:` key).
- Most matchers, snapshots, and `it.each` work unchanged.

> **Grounded:** Vitest documents a [Jest migration guide](https://vitest.dev/guide/migration.html); the API compatibility is deliberate so the switch is mechanical.

---

## Tips

- Mock the LLM in unit tests; reserve real model calls for a small, separate eval/integration suite.
- One behaviour per test; the name should read as a spec ("routes billing query to billing tool").
- `it.each` over copy-pasted near-identical tests.
- `restoreMocks: true` — stop mock state bleeding between tests.
- Test pure logic (parsing, routing, reducers), not the model's outputs — those belong in evals, not unit tests.
- New TS project → Vitest. Existing Jest suite that works → no rush to migrate.
