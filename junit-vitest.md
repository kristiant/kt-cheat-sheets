# JUnit & Vitest

**What they are:** Unit-test frameworks from the same xUnit family — **JUnit 5** for Java/JVM, **Vitest** for JS/TS (Vite-native, Jest-compatible API).

**Why people use them:** Fast, structured automated tests with assertions, fixtures, mocking, and parameterised cases — the safety net for refactoring and CI gates.

**Typically used for:** Unit and integration tests, TDD, regression suites. In TS agent code, Vitest mocks the LLM so you test *your* logic (routing, tool selection, parsing) deterministically without API calls.

> Same concepts, different syntax. This sheet maps them side by side.

---

## Concept map

| Concept | JUnit 5 | Vitest |
|---|---|---|
| Test case | `@Test` | `it()` / `test()` |
| Group | `@Nested` class | `describe()` |
| Setup/teardown | `@BeforeEach` / `@AfterEach` | `beforeEach` / `afterEach` |
| Assertion | `assertEquals(a, b)` | `expect(a).toBe(b)` |
| Parameterised | `@ParameterizedTest` | `it.each()` |
| Mocking | Mockito | `vi.fn()` / `vi.mock()` |
| Expected exception | `assertThrows` | `expect(fn).toThrow()` |

---

## Vitest (TS/JS)

```bash
npm i -D vitest
npx vitest            # watch mode
npx vitest run        # single run (CI)
npx vitest run --coverage
```

### Basics

```ts
import { describe, it, expect, beforeEach } from "vitest";
import { parseIntent } from "../src/intent";

describe("parseIntent", () => {
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
expect(arr).toContain("x");
expect(fn).toThrow("boom");
expect(val).toBeNull();
await expect(promise).resolves.toBe(42);
await expect(promise).rejects.toThrow();
```

### Mocking

```ts
import { vi, expect, it } from "vitest";

const fn = vi.fn().mockReturnValue(7);
fn(1, 2);
expect(fn).toHaveBeenCalledWith(1, 2);

// Mock an entire module
vi.mock("../src/db", () => ({ getUser: vi.fn().mockResolvedValue({ id: 1 }) }));

// Spy on a real object, restore after
const spy = vi.spyOn(console, "log").mockImplementation(() => {});
```

---

## JUnit 5 (Java)

```java
import org.junit.jupiter.api.*;
import static org.junit.jupiter.api.Assertions.*;

class IntentTest {

  @BeforeEach
  void setup() { /* fixtures */ }

  @Test
  void classifiesRefund() {
    assertEquals("refund", IntentParser.parse("I want my money back"));
  }

  @ParameterizedTest
  @CsvSource({ "cancel my plan, cancel", "where's my order, status" })
  void mapsInput(String input, String expected) {
    assertEquals(expected, IntentParser.parse(input));
  }

  @Test
  void rejectsEmpty() {
    assertThrows(IllegalArgumentException.class, () -> IntentParser.parse(""));
  }
}
```

```bash
mvn test           # Maven
./gradlew test     # Gradle
```

Mocking with Mockito:

```java
UserRepo repo = mock(UserRepo.class);
when(repo.findById(1L)).thenReturn(new User(1L));
verify(repo).findById(1L);
```

---

## Advanced — testing agents (Vitest)

### Mock the LLM, assert on your logic

Never hit a real model in unit tests. Inject a fake chat model and assert your graph/tool wiring behaves.

```ts
import { vi, it, expect } from "vitest";
import { FakeListChatModel } from "@langchain/core/utils/testing";

it("routes a billing query to the billing tool", async () => {
  // Deterministic canned responses — no network, no flakiness
  const llm = new FakeListChatModel({ responses: ["billing"] });

  const route = await classify(llm, "my card was charged twice");
  expect(route).toBe("billing");
});
```

> **Grounded:** LangChain ships [`FakeListChatModel`](https://js.langchain.com/docs/integrations/chat/fake) / `FakeChatModel` in `@langchain/core/utils/testing` precisely so you can unit-test chains and graphs deterministically.

### Mock a tool and assert it was called

```ts
const search = vi.fn().mockResolvedValue("result");
const agent = buildAgent({ tools: [makeTool("search", search)], llm });

await agent.invoke({ messages: [userMsg("find X")] });
expect(search).toHaveBeenCalledOnce();
```

### Fake timers (retry/backoff logic)

```ts
import { vi } from "vitest";
vi.useFakeTimers();
const p = retryWithBackoff(flakyFn);
await vi.runAllTimersAsync();        // fast-forward through the delays
vi.useRealTimers();
```

### Snapshot a prompt template

```ts
expect(buildPrompt(context)).toMatchSnapshot();   // catches accidental prompt drift
```

---

## Practical recipes

**Run one test file / one test by name (Vitest):**
```bash
npx vitest run src/intent.test.ts
npx vitest run -t "classifies a refund"
```

**Coverage gate in CI (Vitest):**
```bash
npx vitest run --coverage --coverage.thresholds.lines=80
```

**Run a single test method (JUnit + Maven):**
```bash
mvn test -Dtest=IntentTest#classifiesRefund
```

**Only run changed tests while developing (Vitest):**
```bash
npx vitest --changed
```

**Reset all mocks between tests (Vitest config):**
```ts
// vitest.config.ts → test: { clearMocks: true, restoreMocks: true }
```

---

## Tips

- Mock the LLM in unit tests; reserve real model calls for a small, separate eval/integration suite.
- One behaviour per test; the name should read as a spec ("routes billing query to billing tool").
- `it.each` / `@ParameterizedTest` over copy-pasted near-identical tests.
- `restoreMocks: true` (Vitest) / fresh mocks per test (Mockito) — avoid state bleeding between tests.
- Vitest's API is Jest-compatible — most Jest examples port over by swapping `jest.fn()` → `vi.fn()`.
- Test pure logic (parsing, routing, reducers), not the model's outputs — those belong in evals, not unit tests.
