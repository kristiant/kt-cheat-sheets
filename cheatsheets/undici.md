# undici

**What it is:** Node's HTTP/1.1 client, written from scratch by the Node team — and the engine behind the built-in global `fetch` (Node 18+).

**Why people use it:** Faster than the legacy `http` module, with connection pooling, keep-alive, retries, proxies, and first-class mocking. You get this whenever you call `fetch` in Node; using undici directly unlocks the rest.

**Typically used for:** High-throughput API clients, server-to-server calls, proxy/retry control, and intercepting `fetch` in tests.

> Node-only (not browsers). For the runtime see [nodejs.md](nodejs.md).

---

## `request` vs `fetch`

| | `undici.request()` | `fetch` (global, = undici) |
|---|---|---|
| API | Node-native, low-level | WHATWG standard |
| Returns | `{ statusCode, headers, body, trailers }` | `Response` |
| Body | you must consume/dump it | managed by `Response` |
| Speed | fastest | slightly more overhead |

Use `fetch` for portability and convenience; use `request` for hot paths and fine control. **Dispatchers (pooling, retry, proxy) configure both.**

---

## request — read a body

```ts
import { request } from "undici";

const { statusCode, headers, body } = await request("https://api.example.com/users");
const users = await body.json();        // also: .text(), .arrayBuffer(), .blob()
```

> You **must** consume the body (or call `body.dump()`) — an unread body holds the socket open. This is the #1 undici footgun.

POST with JSON:

```ts
const { body } = await request("https://api.example.com/users", {
  method: "POST",
  headers: { "content-type": "application/json" },
  body: JSON.stringify({ name: "Ada" }),
});
```

Stream the response:

```ts
const { body } = await request(url);
for await (const chunk of body) { /* Buffer chunks */ }
```

---

## Connection management — Agent / Pool / Client

A **dispatcher** decides how requests are routed and pooled. `setGlobalDispatcher` makes `fetch` and `request` use it.

```ts
import { Agent, setGlobalDispatcher } from "undici";

setGlobalDispatcher(new Agent({
  connections: 128,          // max sockets per origin
  keepAliveTimeout: 10_000,  // reuse idle sockets
  pipelining: 1,             // requests per socket before response (1 = off)
}));

await fetch("https://api.example.com/x");   // now uses the pooled agent
```

- **`Agent`** — pools across *many* origins (default global dispatcher).
- **`Pool`** — pooled connections to *one* origin.
- **`Client`** — a *single* connection to one origin (lowest level).

Per-call dispatcher (don't touch the global):

```ts
const pool = new Pool("https://api.example.com");
await request("https://api.example.com/x", { dispatcher: pool });
await fetch("https://api.example.com/x", { dispatcher: pool });
```

---

## Interceptors — retry, redirect, decompress

Compose behaviour onto a dispatcher with `.compose(...)`.

```ts
import { Agent, interceptors, setGlobalDispatcher } from "undici";

const agent = new Agent().compose(
  interceptors.retry({ maxRetries: 3, minTimeout: 500 }),   // exponential backoff on 5xx/network
  interceptors.redirect({ maxRedirections: 3 }),
);
setGlobalDispatcher(agent);
```

Built-in interceptors: `retry`, `redirect`, `dns` (DNS caching), `cache`, `decompress`, `dump`. `RetryAgent` is the standalone equivalent of the retry interceptor.

---

## Proxies

```ts
import { ProxyAgent, EnvHttpProxyAgent, setGlobalDispatcher } from "undici";

setGlobalDispatcher(new ProxyAgent("http://proxy:8080"));      // explicit proxy
setGlobalDispatcher(new EnvHttpProxyAgent());                  // respects HTTP(S)_PROXY / NO_PROXY
```

> Node's global `fetch` ignores `HTTP_PROXY` env vars by default — `EnvHttpProxyAgent` is how you make it honour them.

---

## Testing — MockAgent

Intercept `fetch`/`request` without real network calls.

```ts
import { MockAgent, setGlobalDispatcher } from "undici";

const mock = new MockAgent();
setGlobalDispatcher(mock);
mock.disableNetConnect();                       // fail on any un-mocked request

mock.get("https://api.example.com")
  .intercept({ path: "/users", method: "GET" })
  .reply(200, [{ id: 1, name: "Ada" }]);

const res = await fetch("https://api.example.com/users");   // returns the mock
// in tests: await mock.assertNoPendingInterceptors();
```

---

## Practical recipes

**Pool connections to one API (faster repeated calls):**
```ts
const pool = new Pool("https://api.example.com", { connections: 64 });
await request("https://api.example.com/x", { dispatcher: pool });
```

**Global retry + redirect for all fetch calls:**
```ts
setGlobalDispatcher(new Agent().compose(
  interceptors.retry({ maxRetries: 3 }), interceptors.redirect({ maxRedirections: 5 })));
```

**Per-request timeout:**
```ts
await request(url, { headersTimeout: 5_000, bodyTimeout: 10_000 });
// fetch: fetch(url, { signal: AbortSignal.timeout(5_000) })
```

**Discard a body you don't need (free the socket):**
```ts
const { body } = await request(url);
await body.dump();
```

**Route fetch through a proxy from env:**
```ts
setGlobalDispatcher(new EnvHttpProxyAgent());   // honours HTTPS_PROXY / NO_PROXY
```

**Mock all network in a test suite:**
```ts
const mock = new MockAgent(); setGlobalDispatcher(mock); mock.disableNetConnect();
```

---

## Tips

- Always consume or `dump()` a `request()` body — an unread body leaks the socket.
- Set a global `Agent` with `keepAliveTimeout` for server-to-server traffic — socket reuse is the biggest win.
- Dispatchers configure `fetch` and `request` alike; prefer a per-call `dispatcher` over mutating the global in libraries.
- Use `interceptors.retry` (5xx/network only) — don't hand-roll retry loops; it backs off and respects idempotency.
- `EnvHttpProxyAgent` to make global `fetch` respect proxy env vars (it doesn't by default).
- `MockAgent` + `disableNetConnect()` for hermetic tests of code that calls `fetch`.
- Set `headersTimeout`/`bodyTimeout` (or an `AbortSignal`) — undici won't hang forever, but defaults are generous.
