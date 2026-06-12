# TypeScript — Idiomatic Patterns

**What it is:** The type-level patterns that make TS code safe and self-documenting — discriminated unions, type guards, `satisfies`, generics, and inference tricks.

**Why people use them:** They push errors to compile time and let the compiler *narrow* types for you, so you write fewer runtime checks and get exhaustive coverage of cases.

**Typically used for:** Modelling state machines, API responses, agent/tool definitions, and anywhere "this value is one of N shapes" needs to be enforced.

> For compiler/`tsconfig` setup see [typescript.md](typescript.md). This sheet is patterns.

---

## Discriminated unions (model "one of N shapes")

A shared literal `kind` field lets the compiler narrow which variant you have. The backbone of typed state machines and agent events.

```ts
type AgentEvent =
  | { kind: "tool_call"; tool: string; args: unknown }
  | { kind: "message";   text: string }
  | { kind: "error";     error: Error };

function handle(e: AgentEvent) {
  switch (e.kind) {
    case "tool_call": return run(e.tool, e.args);   // e.args is in scope, e.text is not
    case "message":   return send(e.text);
    case "error":     return log(e.error);
  }
}
```

### Exhaustiveness check

Force a compile error if a new variant is added but unhandled:

```ts
function assertNever(x: never): never {
  throw new Error(`Unhandled: ${JSON.stringify(x)}`);
}

switch (e.kind) {
  case "tool_call": /* ... */ break;
  case "message":   /* ... */ break;
  default: return assertNever(e);   // compile error if "error" case is missing
}
```

---

## Type guards & narrowing

```ts
// User-defined guard
function isToolCall(e: AgentEvent): e is Extract<AgentEvent, { kind: "tool_call" }> {
  return e.kind === "tool_call";
}

// `in` narrowing
if ("error" in e) { /* e has error */ }

// Narrow unknown (e.g. JSON from an API)
function isUser(x: unknown): x is User {
  return typeof x === "object" && x !== null && "id" in x;
}
```

---

## `satisfies` — validate without widening

`satisfies` checks a value against a type **while keeping its narrow literal type**. Use it over `: Type` annotations when you want the precise inferred type back.

```ts
const config = {
  model: "gpt-4o",
  temperature: 0,
} satisfies LLMConfig;

config.model;   // type is "gpt-4o" (literal), not string — still validated against LLMConfig
```

Contrast:

```ts
const a: LLMConfig = { model: "gpt-4o" };  // a.model widens to string
const b = { model: "gpt-4o" } satisfies LLMConfig;  // b.model stays "gpt-4o"
```

---

## `as const` — freeze literals

```ts
const ROLES = ["admin", "user", "guest"] as const;
type Role = (typeof ROLES)[number];        // "admin" | "user" | "guest"

const tool = { name: "search", schema: {...} } as const;  // deeply readonly, literal types
```

---

## Deriving types from values (don't repeat yourself)

```ts
const defaults = { retries: 3, timeout: 5000 };
type Settings = typeof defaults;            // { retries: number; timeout: number }

type Keys = keyof Settings;                 // "retries" | "timeout"
type RetryType = Settings["retries"];       // number
```

With Zod (single source of truth for runtime + compile-time — see [zod.md](zod.md)):

```ts
import { z } from "zod";
const ToolInput = z.object({ city: z.string() });
type ToolInput = z.infer<typeof ToolInput>;  // { city: string }
```

---

## Utility types (transform existing types)

```ts
type PartialUser  = Partial<User>;            // all optional
type RequiredUser = Required<User>;
type PublicUser   = Omit<User, "passwordHash">;
type Creds        = Pick<User, "email" | "passwordHash">;
type ById         = Record<string, User>;
type Result       = ReturnType<typeof fetchUser>;
type Arg          = Parameters<typeof fetchUser>[0];
type Resolved     = Awaited<ReturnType<typeof fetchUser>>;  // unwrap Promise
```

---

## Generics that infer

Let the caller's types flow through — don't force them to annotate.

```ts
function first<T>(arr: T[]): T | undefined {
  return arr[0];
}
first([1, 2, 3]);          // T inferred as number

// Constrain a generic
function pluck<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}
pluck({ id: 1, name: "x" }, "name");   // return type: string
```

---

## Result type (errors as values)

Avoid throw/catch for expected failures — model them in the type so callers must handle both.

```ts
type Result<T, E = Error> =
  | { ok: true;  value: T }
  | { ok: false; error: E };

async function callTool(name: string): Promise<Result<string>> {
  try { return { ok: true, value: await invoke(name) }; }
  catch (e) { return { ok: false, error: e as Error }; }
}

const r = await callTool("search");
if (r.ok) use(r.value); else recover(r.error);   // both paths enforced
```

---

## Branded types (nominal typing)

Stop mixing up two `string`s (a `UserId` vs an `OrgId`) that are structurally identical.

```ts
type UserId = string & { readonly __brand: "UserId" };
const UserId = (s: string) => s as UserId;

function getUser(id: UserId) { /* ... */ }
getUser(UserId("u_1"));   // ok
getUser("u_1");           // compile error — plain string rejected
```

---

## Practical recipes

**Exhaustive event/state handler (agent loop):**
```ts
switch (state.kind) { /* cases */ default: return assertNever(state); }
```

**Type a function's config from a Zod schema:**
```ts
type Config = z.infer<typeof ConfigSchema>;   // runtime validation + static type, one source
```

**Make every field deeply readonly:**
```ts
const frozen = data as const;
```

**Pick a subset type for a public API response:**
```ts
type PublicUser = Omit<User, "passwordHash" | "internalNotes">;
```

**Narrow an `unknown` API payload before use:**
```ts
if (!isUser(json)) throw new Error("bad shape");
json.id;   // now typed
```

---

## Tips

- Reach for **discriminated unions + `assertNever`** for anything with a fixed set of cases — the compiler then enforces you handle all of them.
- Prefer `satisfies` over `: Type` when you want validation *and* the narrow inferred type.
- Derive types from values/schemas (`typeof`, `z.infer`) instead of hand-writing parallel definitions that drift.
- Avoid `any`; use `unknown` + a type guard at boundaries (API responses, JSON, tool outputs).
- Result types make expected failures explicit; reserve `throw` for truly exceptional cases.
- Let generics infer — if callers must pass type arguments, the signature is usually wrong.
