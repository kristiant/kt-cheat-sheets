# Zod

**What it is:** A TypeScript-first schema validation library for checking unknown data at runtime.

**Why people use it:** It turns untrusted inputs into validated, typed values without duplicating separate TypeScript types.

**Typically used for:** API request bodies, query params, environment variables, form data, config files, webhooks, and external API responses.

> This sheet uses Zod 4. Source: https://zod.dev/v4

---

## Mental model

The **schema is the single source of truth** — define the shape once, get runtime validation *and* the static type from it, so they can never drift.

```ts
const UserSchema = z.object({ id: z.string(), age: z.number() });
type User = z.infer<typeof UserSchema>;   // { id: string; age: number } — derived, not duplicated

UserSchema.parse(data);       // → typed value, THROWS on invalid
UserSchema.safeParse(data);   // → { success, data | error } — no throw
```

**Naming convention:** suffix the schema with `Schema` (`UserSchema`) and give the inferred type the plain name (`User`). The suffix is what lets the value and the type coexist without clashing.

"Parse, don't validate": at every boundary (API body, env, webhook, LLM output) turn `unknown` into a typed value and fail loudly if it doesn't fit.

---

## Setup

```bash
npm i zod
```

```ts
import * as z from "zod";
```

Use TypeScript `strict` mode:

```jsonc
{
  "compilerOptions": {
    "strict": true
  }
}
```

---

## Most common APIs

```ts
schema.parse(input);              // returns data or throws ZodError
schema.safeParse(input);          // returns { success, data } or { success, error }
schema.parseAsync(input);         // for async refinements/transforms
schema.safeParseAsync(input);     // async non-throwing parse

type User = z.infer<typeof UserSchema>;   // output type
type In = z.input<typeof UserSchema>;      // input type
type Out = z.output<typeof UserSchema>;    // output type after transforms
```

> Source: Zod's basic usage docs cover `parse`, `safeParse`, async parsing, and type inference: https://zod.dev/basics

## Basic schema

```ts
const UserSchema = z.object({
  id: z.uuid(),
  name: z.string().min(1),
  email: z.email(),
  age: z.number().int().nonnegative().optional(),
});

type User = z.infer<typeof UserSchema>;

const user = UserSchema.parse({
  id: "550e8400-e29b-41d4-a716-446655440000",
  name: "Ada",
  email: "ada@example.com",
});
```

## Safe parsing

```ts
const result = UserSchema.safeParse(input);

if (!result.success) {
  console.error(z.prettifyError(result.error));   // human-readable summary
  throw new Error("Invalid user");
}

result.data.email; // typed
```

## Primitives

```ts
z.string();
z.number();
z.boolean();
z.bigint();
z.date();
z.symbol();
z.undefined();
z.null();
z.any();
z.unknown();
z.never();
```

## Strings

```ts
z.string().min(1);
z.string().max(255);
z.string().length(2);
z.string().trim();
z.string().toLowerCase();
z.email();
z.url();
z.uuid();
z.iso.datetime();
z.string().regex(/^[a-z0-9-]+$/);
```

## Numbers

```ts
z.number().int();
z.number().positive();
z.number().nonnegative();
z.number().min(0).max(100);
z.number().finite();
z.coerce.number(); // "42" -> 42
```

## Objects

These are Zod schema methods — each returns a **new schema** (the original is unchanged), the same way `Array.prototype.map` returns a new array.

```ts
const BaseUserSchema = z.object({ id: z.uuid(), name: z.string() });

BaseUserSchema.pick({ id: true });            // keep only listed keys   → { id }
BaseUserSchema.omit({ id: true });            // drop listed keys        → { name }
BaseUserSchema.partial();                     // make all keys optional  → { id?, name? }
BaseUserSchema.required();                    // make all keys required
BaseUserSchema.extend({ email: z.email() });  // add (or override) keys  → { id, name, email }
```

### Unknown keys

The question these answer: what happens to input keys that *aren't* in the schema. By default Zod **strips** them:

```ts
z.object({ name: z.string() }).parse({ name: "Ada", extra: 1 });
// → { name: "Ada" }    (extra is silently dropped)
```

To change that:

```ts
z.strictObject({ name: z.string() });   // error if any unknown key is present
z.looseObject({ name: z.string() });    // keep unknown keys as-is
```

> In Zod 4 these replace the deprecated `.strict()` / `.passthrough()` methods (which still work for now).

## Arrays, tuples, records

```ts
z.array(z.string());
z.string().array();
z.array(z.number()).min(1).max(10);
z.tuple([z.string(), z.number()]);
z.record(z.string(), z.number());
```

## Unions and enums

```ts
const RoleSchema = z.enum(["admin", "member", "viewer"]);

const IdSchema = z.union([z.string(), z.number()]);

const EventSchema = z.discriminatedUnion("type", [
  z.object({ type: z.literal("created"), id: z.uuid() }),
  z.object({ type: z.literal("deleted"), id: z.uuid(), reason: z.string() }),
]);
```

## Optional, nullable, nullish

```ts
z.string().optional();   // string | undefined          — key may be missing
z.string().nullable();   // string | null               — value may be null
z.string().nullish();    // string | null | undefined   — = .nullable().optional()
```

Pick by what the source can actually produce:

- **`.optional()`** — the field may be *absent* (omitted JSON key, missing form field).
- **`.nullable()`** — the field is present but its value may be `null` (a SQL `NULL` column).
- **`.nullish()`** — either case: present-but-`null` *or* absent. Common for DB rows and partial API payloads.

### Defaults vs null

`.default()` only substitutes for `undefined` — **not `null`**:

```ts
z.string().default("x").parse(undefined);          // "x"
z.string().nullable().default("x").parse(null);    // null  ← default does NOT fire
```

To treat `null` as the default too, map it explicitly:

```ts
z.string().nullish().transform((v) => v ?? "x");   // null AND undefined → "x"
```

`.catch()` is different again — it supplies a fallback when validation *fails entirely*:

```ts
z.number().catch(0).parse("not a number");         // 0
```

## Coercion

`z.coerce.X()` runs the matching JS constructor on the input *before* validating — useful for query strings, env vars, and form data, which arrive as strings.

| Schema | Runs | Example (input → output) |
|---|---|---|
| `z.coerce.string()` | `String(x)` | `42` → `"42"` |
| `z.coerce.number()` | `Number(x)` | `"42"` → `42`, `""` → `0` |
| `z.coerce.boolean()` | `Boolean(x)` | `"false"` → `true` (!), `""` → `false` |
| `z.coerce.date()` | `new Date(x)` | `"2024-01-01"` → `Date` |

```ts
const QuerySchema = z.object({
  page: z.coerce.number().int().positive().default(1),   // "2" → 2
});
QuerySchema.parse({ page: "2" });
```

### `z.coerce.boolean()` is a trap

It's just `Boolean(x)`, so **any non-empty string is `true`** — including `"false"`, `"0"`, and `"no"`:

```ts
z.coerce.boolean().parse("false");   // true  ❌
z.coerce.boolean().parse("");        // false
```

Use **`z.stringbool()`** (Zod 4) for real string→boolean parsing:

```ts
const flag = z.stringbool();   // "true"/"1"/"yes"/"on"/"y"/"enabled" → true
                               // "false"/"0"/"no"/"off"/"n"/"disabled" → false (case-insensitive)
flag.parse("false");   // false ✅
flag.parse("FALSE");   // false
flag.parse("maybe");   // throws ZodError
```

> Coercion widens the input type to `unknown` — `z.input` of a coerced field is `unknown`, not `string`.

## Refinements

```ts
const PasswordSchema = z.string()
  .min(12)
  .refine((value) => /[A-Z]/.test(value), {
    error: "Password must contain an uppercase letter",
  });
```

Cross-field validation:

```ts
const SignupSchema = z.object({
  password: z.string().min(12),
  confirmPassword: z.string(),
}).refine((data) => data.password === data.confirmPassword, {
  path: ["confirmPassword"],
  error: "Passwords do not match",
});
```

## Transforms

```ts
const SlugSchema = z.string()
  .trim()
  .toLowerCase()
  .transform((value) => value.replace(/\s+/g, "-"));

SlugSchema.parse(" Hello World "); // "hello-world"
```

Input and output types can differ:

```ts
const LengthSchema = z.string().transform((value) => value.length);

type LengthInput = z.input<typeof LengthSchema>;   // string
type LengthOutput = z.output<typeof LengthSchema>; // number
```

## Async validation

```ts
const UniqueEmailSchema = z.email().refine(
  async (email) => !(await userExists(email)),
  { error: "Email is already taken" },
);

await UniqueEmailSchema.parseAsync("ada@example.com");
```

---

## Error handling

Zod 4 moved error formatting to **top-level functions** (the old `error.flatten()` / `error.format()` methods are deprecated):

```ts
const result = UserSchema.safeParse(input);

if (!result.success) {
  z.flattenError(result.error);   // { formErrors, fieldErrors } — flat, for forms
  z.treeifyError(result.error);   // nested object mirroring the schema
  z.prettifyError(result.error);  // human-readable string
}
```

```ts
try {
  UserSchema.parse(input);
} catch (error) {
  if (error instanceof z.ZodError) {
    console.error(error.issues);   // array of issues — code, path, message
  }
}
```

Custom error messages — pass a string, or `{ error }` (Zod 4 replaced `{ message }`):

```ts
const NameSchema = z.string().min(1, "Name is required");          // string shorthand
const AgeSchema = z.number().int({ error: "Age must be whole" });  // object form
```

---

## Practical recipes

**Validate environment variables once at startup:**
```ts
const EnvSchema = z.object({
  NODE_ENV: z.enum(["development", "test", "production"]).default("development"),
  PORT: z.coerce.number().int().positive().default(3000),
  DATABASE_URL: z.url(),
});

export const env = EnvSchema.parse(process.env);
```

**Validate an API request body:**
```ts
const CreateUserSchema = z.object({
  name: z.string().min(1),
  email: z.email(),
});

const body = CreateUserSchema.parse(await request.json());
```

**Validate query params from `URLSearchParams`:**
```ts
const SearchQuerySchema = z.object({
  q: z.string().min(1),
  page: z.coerce.number().int().positive().default(1),
});

const query = SearchQuerySchema.parse(Object.fromEntries(url.searchParams));
```

**Validate JSON from `fetch`:**
```ts
const TodoSchema = z.object({
  id: z.number(),
  title: z.string(),
  completed: z.boolean(),
});

const res = await fetch("https://jsonplaceholder.typicode.com/todos/1");
const todo = TodoSchema.parse(await res.json());
```

**Make an update/patch schema from a create schema:**
```ts
const CreatePostSchema = z.object({
  title: z.string().min(1),
  body: z.string().min(1),
});

const UpdatePostSchema = CreatePostSchema.partial();   // all fields optional; wrap in z.strictObject to reject unknown keys
```

**Require at least one field in a patch object:**
```ts
const UpdatePostSchema = CreatePostSchema.partial().refine(
  (value) => Object.keys(value).length > 0,
  { error: "At least one field is required" },
);
```

**Strip unknown keys for public output:**
```ts
const PublicUserSchema = z.object({
  id: z.uuid(),
  name: z.string(),
});

return PublicUserSchema.parse(databaseUser);
```

**Accept either one value or an array:**
```ts
const StringListSchema = z.union([z.string(), z.array(z.string())])
  .transform((value) => Array.isArray(value) ? value : [value]);
```

**Validate a discriminated webhook payload:**
```ts
const WebhookSchema = z.discriminatedUnion("event", [
  z.object({ event: z.literal("user.created"), userId: z.uuid() }),
  z.object({ event: z.literal("user.deleted"), userId: z.uuid(), reason: z.string() }),
]);

const payload = WebhookSchema.parse(await request.json());
```

**Return API-friendly validation errors:**
```ts
const result = Schema.safeParse(input);

if (!result.success) {
  return Response.json({ errors: z.flattenError(result.error) }, { status: 422 });
}
```

---

## Advanced

### Brand IDs so strings don't mix

```ts
const UserIdSchema = z.uuid().brand<"UserId">();
const OrgIdSchema = z.uuid().brand<"OrgId">();

type UserId = z.infer<typeof UserIdSchema>;
type OrgId = z.infer<typeof OrgIdSchema>;
```

### Recursive schemas

Zod 4 supports recursion with a **getter** — the type is inferred automatically, no manual annotation or `z.lazy`:

```ts
const CategorySchema = z.object({
  name: z.string(),
  get children() {
    return z.array(CategorySchema);
  },
});

type Category = z.infer<typeof CategorySchema>;   // { name: string; children: Category[] }
```

### Generate JSON Schema

```ts
const jsonSchema = z.toJSONSchema(UserSchema);
```

> Zod 4 includes JSON Schema conversion in the core docs: https://zod.dev/json-schema

### Use Zod Mini for tighter bundles

```ts
import * as z from "zod/mini";
```

> Zod 4 documents separate `zod`, `zod/mini`, and `zod/v4/core` packages for app and library use.

---

## Tips

- Parse only at system boundaries: HTTP, env, config, files, queues, and third-party APIs.
- Use `safeParse` for request handling; use `parse` when invalid data should crash fast.
- Export schemas and infer types from them; don't hand-write duplicate interfaces.
- Suffix schemas with `Schema` (`const UserSchema`) and infer the plain type (`type User`) — keeps value and type names from clashing.
- Use `z.coerce.*` for query strings and env vars, because they arrive as strings.
- Use `z.stringbool()`, not `z.coerce.boolean()`, for `"true"`/`"false"` strings — `coerce.boolean` treats `"false"` as `true`.
- Remember `.default()` fires only on `undefined`; for `null` use `.nullish().transform()` or `.catch()`.
- Use discriminated unions for event/webhook payloads; they narrow cleanly in TypeScript.
- Format errors with `z.flattenError` (forms), `z.treeifyError` (nested), or `z.prettifyError` (logs) — not the deprecated `.flatten()`/`.format()`.
