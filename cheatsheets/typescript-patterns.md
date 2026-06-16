# TypeScript — Idiomatic Patterns

**What it is:** The type-level patterns that make TS code safe and self-documenting — discriminated unions, `satisfies`, inference from values, mapped/conditional types, and template-literal types.

**Why people use them:** They push errors to compile time and let the compiler *derive* and *narrow* types for you — so a runtime value (a schema, a builder) and its static type stay in sync from one source.

**Typically used for:** Modelling state machines, API responses, ORM/DML schemas, and SDK surfaces where the types must follow the data exactly.

> Compiler/`tsconfig` setup is in [typescript.md](typescript.md). This sheet is patterns.

---

## Everyday patterns

### Discriminated unions + exhaustiveness

A shared literal `kind` lets the compiler narrow the variant. Backbone of typed state machines and event handling.

```ts
type AgentEvent =
  | { kind: "tool_call"; tool: string; args: unknown }
  | { kind: "message";   text: string }
  | { kind: "error";     error: Error };

function assertNever(x: never): never {
  throw new Error(`Unhandled: ${JSON.stringify(x)}`);
}

function handle(e: AgentEvent) {
  switch (e.kind) {
    case "tool_call": return run(e.tool, e.args);   // e.args in scope here
    case "message":   return send(e.text);
    case "error":     return log(e.error);
    default:          return assertNever(e);          // compile error if a case is missed
  }
}
```

### Compose with `&` (intersection)

Share a common field set across many types by intersecting it in, instead of repeating fields or forcing an `extends` chain.

```ts
interface ContentMetadata {
  providerMetadata?: Record<string, unknown>;
}

type Citation = ContentMetadata & {   // = the shared fields + these
  type: "citation";
  url?: string;
  title?: string;
};
```

### Shared fields + a discriminated union

Intersect a shared type with a parenthesised union: every value carries the common fields **and** is exactly one variant. Narrowing on the discriminant still works.

```ts
type StreamChunk = ContentMetadata & (
  | { type: "text-delta"; id: string; delta: string }
  | { type: "tool-call"; toolCallId: string; input: unknown }
  | { type: "finish"; finishReason: FinishReason; usage?: TokenUsage }
  | { type: "error"; error: unknown }
);

function handle(c: StreamChunk) {
  c.providerMetadata;                  // available on every variant (from the intersection)
  if (c.type === "text-delta") c.delta; // narrowed
}
```

### Type guards & narrowing

```ts
function isUser(x: unknown): x is User {              // narrow unknown at boundaries
  return typeof x === "object" && x !== null && "id" in x;
}
if ("error" in e) { /* e has error */ }              // `in` narrowing
```

### Assertion functions — `asserts x is T`

Throw-on-invalid *and* narrow the variable for the rest of the scope. Prefer over a non-null `!` or re-checking.

```ts
function assertIsUser(x: unknown): asserts x is User {
  if (!isUser(x)) throw new Error("not a User");
}
assertIsUser(data);
data.id;                                              // ✅ narrowed to User, no `!`

// assert non-null (common in repos/services)
function assertDefined<T>(v: T | null | undefined, msg: string): asserts v is T {
  if (v == null) throw new Error(msg);
}
```

> Prefer type guards / assertion functions over `as` casts — `as` silences the compiler without checking. Reserve `as` for test code. (See [practices/typescript-error-handling.md](../practices/typescript-error-handling.md) for static `asserts` on error classes.)

### `satisfies` — validate without widening

Checks a value against a type while **keeping its narrow literal type** — unlike `: Type`, which widens.

```ts
const config = { model: "gpt-4o", temperature: 0 } satisfies LLMConfig;
config.model;   // type is "gpt-4o" (literal), still validated against LLMConfig
```

### `as const`

```ts
const ROLES = ["admin", "user", "guest"] as const;
type Role = (typeof ROLES)[number];                  // "admin" | "user" | "guest"
```

### Utility types

```ts
type PublicUser = Omit<User, "passwordHash">;
type Creds      = Pick<User, "email" | "passwordHash">;
type ById       = Record<string, User>;
type Resolved   = Awaited<ReturnType<typeof fetchUser>>;
```

### Generics that infer

```ts
function pluck<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}
pluck({ id: 1, name: "x" }, "name");   // return type: string, inferred
```

### Optional generic with a default

A defaulted type param is a precision *hook*: the loose default keeps most call sites simple, while callers who know the shape opt into stricter typing — without changing the base type.

```ts
type TokenUsage<T extends Record<string, unknown> = Record<string, unknown>> = {
  totalTokens: number;
  metadata?: T;
};

type Loose = TokenUsage;                          // metadata?: Record<string, unknown>
type Tight = TokenUsage<{ provider: "openai" }>;  // metadata?: { provider: "openai" }
```

### Result type (errors as values)

```ts
type Result<T, E = Error> = { ok: true; value: T } | { ok: false; error: E };

const r = await callTool("search");
if (r.ok) use(r.value); else recover(r.error);       // both paths enforced
```

### Branded types (nominal)

```ts
type UserId = string & { readonly __brand: "UserId" };
const UserId = (s: string) => s as UserId;
getUser(UserId("u_1"));   // plain string rejected
```

### `interface` vs `type`

```ts
interface User { id: string; name: string }   // object shape — extendable / implementable
type Status = "active" | "archived";           // union — must be `type`
type WithId<T> = T & { id: string };           // intersection / mapped / conditional — `type`
```

Rules of thumb:

- **`interface`** for object shapes you extend, implement, or expose as a public API — cleaner extends, better error messages, declaration merging.
- **`type`** for anything `interface` can't express: unions, intersections, tuples, mapped/conditional, primitives, function types.
- Plain object needing neither? Either works — pick one and stay consistent. (Declaration merging is `interface`-only; a feature for public types, a footgun for app types.)

### `readonly` + inline JSDoc

Mark properties immutable-after-construction with `readonly`; put JSDoc on the *individual member* (e.g. `@internal` to hide one field), not the whole interface.

```ts
interface BuiltGuardrail {
  readonly name: string;
  readonly strategy: "block" | "redact" | "warn";
  /** @internal */ readonly _config: Record<string, unknown>;
}
```

### Fluent builders — return `this`

Methods that return the polymorphic `this` type chain together — and a subclass keeps its own type through the chain (unlike returning the concrete class name).

```ts
class Guardrail {
  private guardType?: GuardrailType;
  private strategyType?: GuardrailStrategy;

  type(t: GuardrailType): this { this.guardType = t; return this; }
  strategy(s: GuardrailStrategy): this { this.strategyType = s; return this; }
}

new Guardrail().type("pii").strategy("block");   // each step returns `this` → chainable
```

---

## Advanced

These show up in ORM/SDK/validation library internals, where types must follow runtime data exactly.

### `Prettify<T>` — flatten intersections for readable types

Intersections (`A & B & C`) show as a tangle on hover. This collapses them to one flat object — purely cosmetic, but huge for DX on derived types.

```ts
type Prettify<T> = { [K in keyof T]: T[K] } & {};
```

### Infer a static type from a runtime schema

The headline pattern: define a schema **once** as a value, then derive its type with `infer` — runtime shape and compile-time type can never drift. Zod is the canonical example:

```ts
import { z } from "zod";

const Currency = z.object({
  code: z.string(),
  symbol: z.string(),
  decimals: z.number().nullable(),
});

type Currency = z.infer<typeof Currency>;
// { code: string; symbol: string; decimals: number | null }
```

The same technique on your own builder — capture the schema in a generic, unwrap it with a conditional `infer`:

```ts
interface Schema<T> { __type: T }
type InferType<S> = S extends Schema<infer T> ? T : never;
```

> See [zod.md](zod.md) — one source of truth for runtime validation *and* static types.

### Strip auto-managed fields for input DTOs

Derive a "create input" type from an entity by `Omit`-ting the fields the system manages.

```ts
type Managed = "created_at" | "updated_at" | "deleted_at";
type CreateInput<T> = Omit<T, Managed>;
```

### Mapped type that transforms a config object

Walk every key of an input type and branch on what each value *is* (`infer` + conditional). Turns a config map into a derived config map.

```ts
type ToConfig<T> = {
  [K in keyof T]: T[K] extends Schema<infer U>
    ? U                                    // a schema → its inferred type
    : T[K] extends Constructor<any>
    ? InstanceType<T[K]>                   // a class → its instance type
    : T[K] extends { model: infer M }      // an object → pull out its `model`
    ? M
    : never;
};
```

### Factory typing — `Constructor` / `InstanceType`

Type functions that build classes (mixins, service factories).

```ts
type Constructor<T> = new (...args: any[]) => T;

function withTimestamps<T extends Constructor<{}>>(Base: T) {
  return class extends Base {
    createdAt = new Date();
  };
}
type Built = InstanceType<ReturnType<typeof withTimestamps>>;
```

### Template-literal types — parse strings at the type level

Conditional types recurse over string literals with `infer`. Common in SDKs that derive method names (`listProducts`, `retrieveUser`) from entity names entirely in the type system.

```ts
type Pluralize<S extends string> =
  S extends `${infer R}y`  ? `${R}ies`
  : S extends `${string}s` ? S
  : `${S}s`;

type T1 = Pluralize<"category">;   // "categories"
type T2 = Pluralize<"address">;    // "addresses"
```

### Depth-limited recursion (the tuple-counter trick)

Recursive types over nested objects loop forever without a bound. Index a tuple to "decrement" a depth counter and stop — the standard way to cap how deep a recursive type (e.g. a field-path expander) descends.

```ts
type Decrement = [never, 0, 1, 2, 3, 4];           // Decrement[3] = 2, Decrement[1] = 0

type Paths<T, Depth extends number = 3> = Depth extends never
  ? never                                          // hit the floor → stop
  : T extends object
  ? { [K in keyof T]: /* recurse with */ Paths<T[K], Decrement[Depth]> }[keyof T]
  : never;
```

### Object → union of its leaves via `[keyof T]`

Indexing a mapped type by `keyof T` collapses it into a **union** of the values — the trick behind extracting field paths.

```ts
type ValuesOf<T> = { [K in keyof T]: T[K] }[keyof T];
type U = ValuesOf<{ a: "x"; b: "y" }>;   // "x" | "y"
```

---

## Practical recipes

**Exhaustive handler (agent/state loop):**
```ts
switch (s.kind) { /* cases */ default: return assertNever(s); }
```

**Derive a type from a schema instead of hand-writing it:**
```ts
type T = z.infer<typeof Schema>;
```

**Make a create-input type from an entity:**
```ts
type CreateInput = Omit<z.infer<typeof Schema>, "created_at" | "updated_at" | "deleted_at">;
```

**Flatten an ugly intersection on hover:**
```ts
type Clean = Prettify<A & B & C>;
```

**Validate config but keep literal types:**
```ts
const cfg = { model: "gpt-4o" } satisfies LLMConfig;
```

---

## Tips

- Derive types from values/schemas (`z.infer`, `typeof`) — never hand-maintain a type that parallels a runtime shape; it will drift.
- Wrap derived/intersection types in `Prettify` so consumers see a clean object, not `A & B & {...}`.
- `satisfies` when you want validation *and* the narrow inferred type; `: Type` only when you want to widen.
- Discriminated unions + `assertNever` for any fixed set of cases — the compiler then forces you to handle all of them.
- Intersect shared fields into a discriminated union (`Shared & (A | B)`) so every variant carries them and narrowing still works.
- `interface` for extendable object shapes; `type` for unions/intersections/mapped types. Default a generic param to keep the loose case simple while allowing precision.
- Bound recursive types with a tuple-counter; unbounded recursion errors with "type instantiation is excessively deep".
- `unknown` + a type guard at every boundary (API, JSON, tool output); avoid `any`.
- Reach for template-literal types sparingly — powerful for SDK ergonomics, but slow to compile and hard to debug.
