# Zod

**What it is:** A TypeScript-first schema validation library for checking unknown data at runtime.

**Why people use it:** It turns untrusted inputs into validated, typed values without duplicating separate TypeScript types.

**Typically used for:** API request bodies, query params, environment variables, form data, config files, webhooks, and external API responses.

> This sheet uses Zod 4. Source: https://zod.dev/v4

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

type User = z.infer<typeof User>; // output type
type In = z.input<typeof Schema>; // input type
type Out = z.output<typeof Schema>; // output type after transforms
```

> Source: Zod's basic usage docs cover `parse`, `safeParse`, async parsing, and type inference: https://zod.dev/basics

## Basic schema

```ts
const User = z.object({
  id: z.uuid(),
  name: z.string().min(1),
  email: z.email(),
  age: z.number().int().nonnegative().optional(),
});

type User = z.infer<typeof User>;

const user = User.parse({
  id: "550e8400-e29b-41d4-a716-446655440000",
  name: "Ada",
  email: "ada@example.com",
});
```

## Safe parsing

```ts
const result = User.safeParse(input);

if (!result.success) {
  console.error(result.error.flatten());
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

```ts
const BaseUser = z.object({
  id: z.uuid(),
  name: z.string(),
});

const PublicUser = BaseUser.pick({ id: true, name: true });
const UserPatch = BaseUser.partial();
const StrictUser = BaseUser.strict();
const LooseUser = BaseUser.passthrough();
const UserWithEmail = BaseUser.extend({ email: z.email() });
```

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
const Role = z.enum(["admin", "member", "viewer"]);

const Id = z.union([z.string(), z.number()]);

const Event = z.discriminatedUnion("type", [
  z.object({ type: z.literal("created"), id: z.uuid() }),
  z.object({ type: z.literal("deleted"), id: z.uuid(), reason: z.string() }),
]);
```

## Optional, nullable, defaults

```ts
z.string().optional();             // string | undefined
z.string().nullable();             // string | null
z.string().nullish();              // string | null | undefined
z.string().default("untitled");    // undefined -> "untitled"
z.string().catch("fallback");      // invalid -> "fallback"
```

## Coercion

```ts
const Query = z.object({
  page: z.coerce.number().int().positive().default(1),
  includeArchived: z.coerce.boolean().default(false),
});

Query.parse({ page: "2", includeArchived: "true" });
```

## Refinements

```ts
const Password = z.string()
  .min(12)
  .refine((value) => /[A-Z]/.test(value), {
    message: "Password must contain an uppercase letter",
  });
```

Cross-field validation:

```ts
const Signup = z.object({
  password: z.string().min(12),
  confirmPassword: z.string(),
}).refine((data) => data.password === data.confirmPassword, {
  path: ["confirmPassword"],
  message: "Passwords do not match",
});
```

## Transforms

```ts
const Slug = z.string()
  .trim()
  .toLowerCase()
  .transform((value) => value.replace(/\s+/g, "-"));

Slug.parse(" Hello World "); // "hello-world"
```

Input and output types can differ:

```ts
const Length = z.string().transform((value) => value.length);

type LengthInput = z.input<typeof Length>;   // string
type LengthOutput = z.output<typeof Length>; // number
```

## Async validation

```ts
const UniqueEmail = z.email().refine(
  async (email) => !(await userExists(email)),
  { message: "Email is already taken" },
);

await UniqueEmail.parseAsync("ada@example.com");
```

---

## Error handling

```ts
const result = User.safeParse(input);

if (!result.success) {
  return {
    fieldErrors: result.error.flatten().fieldErrors,
    formErrors: result.error.flatten().formErrors,
  };
}
```

```ts
try {
  User.parse(input);
} catch (error) {
  if (error instanceof z.ZodError) {
    console.error(error.issues);
  }
}
```

Custom messages:

```ts
const Name = z.string().min(1, "Name is required");
const Age = z.number().int("Age must be a whole number");
```

---

## Practical recipes

**Validate environment variables once at startup:**
```ts
const Env = z.object({
  NODE_ENV: z.enum(["development", "test", "production"]).default("development"),
  PORT: z.coerce.number().int().positive().default(3000),
  DATABASE_URL: z.url(),
});

export const env = Env.parse(process.env);
```

**Validate an API request body:**
```ts
const CreateUser = z.object({
  name: z.string().min(1),
  email: z.email(),
});

const body = CreateUser.parse(await request.json());
```

**Validate query params from `URLSearchParams`:**
```ts
const SearchQuery = z.object({
  q: z.string().min(1),
  page: z.coerce.number().int().positive().default(1),
});

const query = SearchQuery.parse(Object.fromEntries(url.searchParams));
```

**Validate JSON from `fetch`:**
```ts
const Todo = z.object({
  id: z.number(),
  title: z.string(),
  completed: z.boolean(),
});

const res = await fetch("https://jsonplaceholder.typicode.com/todos/1");
const todo = Todo.parse(await res.json());
```

**Make an update/patch schema from a create schema:**
```ts
const CreatePost = z.object({
  title: z.string().min(1),
  body: z.string().min(1),
});

const UpdatePost = CreatePost.partial().strict();
```

**Require at least one field in a patch object:**
```ts
const UpdatePost = CreatePost.partial().refine(
  (value) => Object.keys(value).length > 0,
  { message: "At least one field is required" },
);
```

**Strip unknown keys for public output:**
```ts
const PublicUser = z.object({
  id: z.uuid(),
  name: z.string(),
});

return PublicUser.parse(databaseUser);
```

**Accept either one value or an array:**
```ts
const StringList = z.union([z.string(), z.array(z.string())])
  .transform((value) => Array.isArray(value) ? value : [value]);
```

**Validate a discriminated webhook payload:**
```ts
const Webhook = z.discriminatedUnion("event", [
  z.object({ event: z.literal("user.created"), userId: z.uuid() }),
  z.object({ event: z.literal("user.deleted"), userId: z.uuid(), reason: z.string() }),
]);

const payload = Webhook.parse(await request.json());
```

**Return API-friendly validation errors:**
```ts
const result = Schema.safeParse(input);

if (!result.success) {
  return Response.json({ errors: result.error.flatten() }, { status: 422 });
}
```

---

## Advanced

### Brand IDs so strings don't mix

```ts
const UserId = z.uuid().brand<"UserId">();
const OrgId = z.uuid().brand<"OrgId">();

type UserId = z.infer<typeof UserId>;
type OrgId = z.infer<typeof OrgId>;
```

### Recursive schemas

```ts
type Category = {
  name: string;
  children: Category[];
};

const Category: z.ZodType<Category> = z.object({
  name: z.string(),
  children: z.lazy(() => Category.array()),
});
```

### Generate JSON Schema

```ts
const jsonSchema = z.toJSONSchema(User);
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
- Use `z.coerce.*` for query strings and env vars, because they arrive as strings.
- Use discriminated unions for event/webhook payloads; they narrow cleanly in TypeScript.
- Be careful with `z.coerce.boolean()` for user input; test the exact strings your clients send.
