# Node.js

**What it is:** A JavaScript runtime built on V8 that runs JS/TS outside the browser, with APIs for files, networking, processes, and streams.

**Why people use it:** One language across front and back end, a huge package ecosystem (npm), and an event-loop model that handles many concurrent I/O operations efficiently.

**Typically used for:** APIs and servers, CLIs, build tooling, and the runtime for TS agent/LLM apps (LangChain.js, LangGraph.js).

---

## Mental model — the event loop

Node runs your JavaScript on a **single thread**. I/O (files, network, DB, model APIs) is **non-blocking**: it's handed off and the result returns later as a promise/callback, while the thread keeps serving other work. This is why everything is `async`, and why one process handles thousands of concurrent connections.

Don't block the loop with heavy synchronous CPU work — it freezes every request. Offload it (worker threads) or keep it off the hot path. Most LLM-app latency is I/O, which the loop already handles well.

---

## Modules: ESM vs CommonJS

```js
// ESM (modern — set "type": "module" in package.json)
import { readFile } from "node:fs/promises";
export function foo() {}

// CommonJS (legacy default)
const { readFile } = require("node:fs/promises");
module.exports = { foo };
```

- Use **ESM** for new projects. Prefix built-ins with `node:` (`node:fs`, `node:path`).
- Top-level `await` works in ESM only.

---

## Async patterns

```js
// async/await (default choice)
async function load() {
  const data = await readFile("x.json", "utf8");
  return JSON.parse(data);
}

// Parallel — await independent work together
const [a, b] = await Promise.all([fetchA(), fetchB()]);

// First to finish / settle-all
await Promise.race([task(), timeout(5000)]);
const results = await Promise.allSettled(tasks);   // never rejects; inspect each
```

> `Promise.all` rejects on the *first* failure. Use `allSettled` when you need every result regardless — e.g. fanning out N tool calls and tolerating partial failure.

---

## Common built-ins

```js
import { readFile, writeFile, readdir } from "node:fs/promises";
import path from "node:path";
import { fileURLToPath } from "node:url";

const __dirname = path.dirname(fileURLToPath(import.meta.url));   // ESM __dirname
const full = path.join(__dirname, "data", "x.json");

process.env.NODE_ENV;          // env vars
process.argv.slice(2);         // CLI args
process.exit(1);               // exit code
```

### Environment variables

```bash
node --env-file=.env app.js    # built-in .env loading (Node 20.6+) — no dotenv needed
```

```js
const key = process.env.OPENAI_API_KEY;
if (!key) throw new Error("OPENAI_API_KEY missing");   // fail fast on startup
```

---

## Error handling

```js
try {
  await risky();
} catch (err) {
  if (err instanceof TypeError) { /* specific */ }
  throw err;   // re-throw what you can't handle
}

// Catch async leaks so the process doesn't die silently
process.on("unhandledRejection", (e) => { console.error(e); process.exit(1); });
```

---

## Streams (large data, don't buffer it all)

```js
import { createReadStream, createWriteStream } from "node:fs";
import { pipeline } from "node:stream/promises";

await pipeline(
  createReadStream("big.csv"),
  transformStream,
  createWriteStream("out.csv"),
);
```

Stream an LLM response to the client as tokens arrive:

```js
for await (const chunk of await model.stream(prompt)) {
  res.write(chunk.content);   // flush tokens incrementally
}
res.end();
```

---

## Advanced

### CPU-bound work — worker threads

The event loop is single-threaded; offload heavy CPU work (parsing, embeddings math) so it doesn't block I/O.

```js
import { Worker } from "node:worker_threads";
const worker = new Worker("./heavy.js", { workerData: { input } });
worker.on("message", (result) => console.log(result));
```

> For LLM apps, most latency is **I/O** (waiting on the model API), which the event loop already handles well — only reach for workers on genuine CPU hotspots.

### Graceful shutdown

Drain in-flight work before exit (important for queue/agent workers):

```js
process.on("SIGTERM", async () => {
  await server.close();
  await flushPendingJobs();
  process.exit(0);
});
```

### AbortController (cancel/timeout)

```js
const ac = new AbortController();
setTimeout(() => ac.abort(), 5000);
await fetch(url, { signal: ac.signal });          // works with many Node APIs
await model.invoke(prompt, { signal: ac.signal }); // cancel a hung LLM call
```

### package.json scripts

```json
{
  "type": "module",
  "scripts": {
    "dev":   "node --watch --env-file=.env src/index.js",
    "start": "node src/index.js",
    "test":  "vitest run"
  }
}
```

---

## Practical recipes

**Run a TS file directly (Node 22+, no build step):**
```bash
node --experimental-strip-types src/index.ts
```

**Watch + auto-restart on change (built-in, no nodemon):**
```bash
node --watch --env-file=.env src/index.js
```

**Read JSON config safely:**
```js
const cfg = JSON.parse(await readFile("config.json", "utf8"));
```

**Fan out tool calls, tolerate partial failure:**
```js
const results = await Promise.allSettled(tools.map((t) => t.invoke(args)));
const ok = results.filter((r) => r.status === "fulfilled").map((r) => r.value);
```

**Timeout any promise:**
```js
const withTimeout = (p, ms) =>
  Promise.race([p, new Promise((_, rej) => setTimeout(() => rej(new Error("timeout")), ms))]);
```

**Validate required env at boot:**
```js
for (const k of ["OPENAI_API_KEY", "DATABASE_URL"])
  if (!process.env[k]) throw new Error(`Missing env: ${k}`);
```

---

## Tips

- Default to **ESM** + `node:` prefixes; only use CJS for legacy.
- Node 20.6+ loads `.env` natively (`--env-file`) — drop the `dotenv` dependency for simple cases.
- Validate env vars at startup and fail fast, not on first use deep in a request.
- `Promise.all` for "all must succeed", `allSettled` for "tolerate partial failure".
- Pass an `AbortSignal` to cancel slow LLM/HTTP calls instead of leaking hung requests.
- Most LLM-app latency is I/O — the event loop scales it; reserve worker threads for CPU hotspots.
