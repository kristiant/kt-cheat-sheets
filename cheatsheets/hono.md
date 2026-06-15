# Hono

**What it is:** A small TypeScript web framework built on Web Standard `Request`/`Response` APIs.

**Why people use it:** It gives Express-like routing with strong TypeScript support and code that can move between Node.js, Bun, Deno, Cloudflare Workers, and other runtimes.

**Typically used for:** JSON APIs, edge/serverless handlers, small Node.js services, middleware-heavy API gateways, and typed RPC-style backends.

> This sheet focuses on Hono on Node.js with `@hono/node-server`.

---

## Mental model

A handler is `(c) => Response`, where **`c` (Context)** carries the request *and* the response helpers. Hono speaks only **Web-standard `Request`/`Response`** — which is why the same code runs on Node, Bun, Deno, and Workers.

```ts
app.get("/users/:id", (c) => {
  const id = c.req.param("id");     // request in  ─┐
  return c.json({ id });            // response out ─┘  all via `c`
});
```

Middleware wraps handlers via `await next()` — an onion where each layer runs code before and after the next.

---

## Setup

```bash
npm create hono@latest my-api   # choose nodejs
cd my-api
npm i
npm run dev
```

Manual install:

```bash
npm i hono @hono/node-server
npm i -D typescript tsx @types/node
```

```json
{
  "type": "module",
  "scripts": {
    "dev": "tsx watch src/index.ts",
    "start": "node dist/index.js",
    "build": "tsc"
  }
}
```

> Source: Hono's Node.js guide uses `create-hono`, `@hono/node-server`, and requires modern Node 18+ versions: https://hono.dev/docs/getting-started/nodejs

---

## Most common commands

```bash
npm run dev                         # start local dev server
npm run build                       # compile TypeScript
npm start                           # run compiled server
npm i hono @hono/node-server        # install for Node.js
npm i @hono/zod-validator zod       # request validation
```

## Hello world

```ts
// src/index.ts
import { serve } from "@hono/node-server";
import { Hono } from "hono";

const app = new Hono();

app.get("/", (c) => c.text("Hello Hono"));

serve(app);
```

## Routes

```ts
app.get("/users", (c) => c.json([{ id: "u1", name: "Ada" }]));

app.get("/users/:id", (c) => {
  const id = c.req.param("id");
  return c.json({ id });
});

app.post("/users", async (c) => {
  const body = await c.req.json();
  return c.json({ id: crypto.randomUUID(), ...body }, 201);
});

app.delete("/users/:id", (c) => c.body(null, 204));
```

## Request data

```ts
app.get("/search", (c) => {
  const q = c.req.query("q");
  const tags = c.req.queries("tag");
  const userAgent = c.req.header("User-Agent");

  return c.json({ q, tags, userAgent });
});
```

```bash
curl 'http://localhost:3000/search?q=hono&tag=api&tag=node'
```

## Responses

```ts
app.get("/text", (c) => c.text("ok"));
app.get("/json", (c) => c.json({ ok: true }));
app.get("/html", (c) => c.html("<h1>ok</h1>"));
app.get("/redirect", (c) => c.redirect("/json"));

app.get("/headers", (c) => {
  c.header("Cache-Control", "no-store");
  return c.json({ ok: true });
});
```

---

## Middleware

### Built-in middleware

```ts
import { bearerAuth } from "hono/bearer-auth";
import { cors } from "hono/cors";
import { logger } from "hono/logger";
import { secureHeaders } from "hono/secure-headers";

app.use(logger());
app.use(secureHeaders());
app.use("/api/*", cors({ origin: "http://localhost:5173" }));
app.use("/admin/*", bearerAuth({ token: process.env.ADMIN_TOKEN! }));
```

### Custom middleware

```ts
app.use(async (c, next) => {
  const start = performance.now();
  await next();
  console.log(`${c.req.method} ${c.req.path} ${c.res.status} ${performance.now() - start}ms`);
});
```

### Per-route middleware

```ts
import type { MiddlewareHandler } from "hono";

const requireApiKey: MiddlewareHandler = async (c, next) => {
  if (c.req.header("X-API-Key") !== process.env.API_KEY) {
    return c.json({ error: "Unauthorized" }, 401);
  }
  await next();
};

app.get("/private", requireApiKey, (c) => c.json({ ok: true }));
```

---

## Validation with Zod

```bash
npm i zod @hono/zod-validator
```

```ts
import { zValidator } from "@hono/zod-validator";
import * as z from "zod";

const CreateUser = z.object({
  name: z.string().min(1),
  email: z.email(),
});

app.post("/users", zValidator("json", CreateUser), (c) => {
  const user = c.req.valid("json");
  return c.json({ id: crypto.randomUUID(), ...user }, 201);
});
```

Custom validation response:

```ts
app.post(
  "/users",
  zValidator("json", CreateUser, (result, c) => {
    if (!result.success) {
      return c.json({ error: result.error.flatten() }, 422);
    }
  }),
  (c) => c.json(c.req.valid("json"), 201),
);
```

> Source: Hono recommends third-party validators and documents `@hono/zod-validator`: https://hono.dev/docs/guides/validation

---

## Group routes

```ts
const api = new Hono();

api.get("/health", (c) => c.json({ ok: true }));
api.get("/users/:id", (c) => c.json({ id: c.req.param("id") }));

app.route("/api", api);
```

## Error handling

```ts
app.notFound((c) => c.json({ error: "Not found" }, 404));

app.onError((err, c) => {
  console.error(err);
  return c.json({ error: "Internal server error" }, 500);
});
```

## Environment bindings

```ts
type Env = {
  Variables: {
    requestId: string;
  };
};

const app = new Hono<Env>();

app.use(async (c, next) => {
  c.set("requestId", crypto.randomUUID());
  await next();
});

app.get("/", (c) => c.json({ requestId: c.get("requestId") }));
```

## Change the port

```ts
serve({
  fetch: app.fetch,
  port: Number(process.env.PORT ?? 3000),
});
```

## Graceful shutdown

```ts
const server = serve(app);

process.on("SIGTERM", () => {
  server.close((err) => {
    if (err) {
      console.error(err);
      process.exit(1);
    }
    process.exit(0);
  });
});
```

## Static files

```ts
import { fileURLToPath } from "node:url";
import { serveStatic } from "@hono/node-server/serve-static";

app.use(
  "/static/*",
  serveStatic({ root: fileURLToPath(new URL("./", import.meta.url)) }),
);
```

> Hono warns that `root: "./"` resolves from `process.cwd()`, not the source file location.

---

## Advanced

### Export the app for tests and deployment

```ts
// src/app.ts
import { Hono } from "hono";

export const app = new Hono();
app.get("/health", (c) => c.json({ ok: true }));
```

```ts
// src/index.ts
import { serve } from "@hono/node-server";
import { app } from "./app.js";

serve(app);
```

### Test handlers without opening a port

```ts
import { describe, expect, it } from "vitest";
import { app } from "./app";

describe("GET /health", () => {
  it("returns ok", async () => {
    const res = await app.request("/health");
    expect(res.status).toBe(200);
    expect(await res.json()).toEqual({ ok: true });
  });
});
```

### Access raw Node.js request details

```ts
import { serve, type HttpBindings } from "@hono/node-server";
import { Hono } from "hono";

const app = new Hono<{ Bindings: HttpBindings }>();

app.get("/ip", (c) => {
  return c.json({ ip: c.env.incoming.socket.remoteAddress });
});

serve(app);
```

### Type a Hono client from server routes

```ts
// server.ts
import { Hono } from "hono";

export const app = new Hono()
  .get("/users/:id", (c) => c.json({ id: c.req.param("id") }));

export type AppType = typeof app;
```

```ts
// client.ts
import { hc } from "hono/client";
import type { AppType } from "./server";

const client = hc<AppType>("http://localhost:3000");
const res = await client.users[":id"].$get({ param: { id: "u1" } });
```

---

## Practical recipes

**Create a new Node.js Hono API:**
```bash
npm create hono@latest my-api
```

**Add a health check endpoint:**
```ts
app.get("/healthz", (c) => c.json({ ok: true }));
```

**Return a typed 422 validation error:**
```ts
zValidator("json", Schema, (result, c) => {
  if (!result.success) return c.json({ errors: result.error.flatten() }, 422);
});
```

**Protect all `/admin/*` routes with a bearer token:**
```ts
app.use("/admin/*", bearerAuth({ token: process.env.ADMIN_TOKEN! }));
```

**Allow a Vite dev frontend to call your API:**
```ts
app.use("/api/*", cors({ origin: "http://localhost:5173" }));
```

**Read JSON safely after validation:**
```ts
const data = c.req.valid("json");
```

**Serve files from a local `public/` directory:**
```ts
app.use("/public/*", serveStatic({ root: fileURLToPath(new URL("./", import.meta.url)) }));
```

**Run curl against a POST route:**
```bash
curl -i -X POST http://localhost:3000/users \
  -H 'Content-Type: application/json' \
  -d '{"name":"Ada","email":"ada@example.com"}'
```

**Build a small Docker image:**
```dockerfile
FROM node:22-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build
CMD ["node", "dist/index.js"]
```

---

## Tips

- Keep route handlers small; move business logic to plain functions you can test without Hono.
- Use `c.req.valid(...)` after `zValidator`; don't parse the same body twice.
- Put broad middleware before routes, and scoped middleware on path prefixes like `/api/*`.
- Prefer `app.request()` tests for fast handler coverage.
- Hono code is portable when you stick to Web APIs; Node-only APIs reduce that portability.
