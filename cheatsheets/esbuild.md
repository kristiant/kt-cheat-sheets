# esbuild

**What it is:** An extremely fast (Go-based) JS/TS bundler and transpiler. It strips types, bundles modules, minifies — but does **not** type-check.

**Why people use it:** 10–100× faster than older tools, and it's the engine under much of the modern toolchain (tsx, tsup, Lambda bundling). Learn it once; it shows up everywhere.

**Typically used for:** Bundling serverless/Lambda functions, building libraries (via tsup), and fast TS transpilation in dev (via tsx).

> Backend/library focused. Frontend apps usually use Vite (which wraps similar tooling), not esbuild directly.

---

## Do you even need to bundle?

The backend's first question — and often the answer is **no**. It depends on where the code runs:

| Context | Bundle? | Why |
|---|---|---|
| Long-running server (Express/Fastify/Hono on a VM/container) | Usually **no** | Node loads `node_modules` itself; just transpile or run the TS directly |
| Serverless / edge (Lambda, Workers) | **Yes** | A small single-file artifact → faster cold starts, fits size limits |
| Library (npm package) | **Yes** | Ship clean ESM/CJS + types; the consumer bundles the rest |

Bundling earns its keep when a small self-contained artifact matters (cold starts, size caps) or when publishing. A plain server that ships its `node_modules` doesn't need it — unlike the frontend, "no bundler, just transpile" is a normal backend choice.

---

## The ecosystem (esbuild is the engine)

| Tool | Role | Built on esbuild? |
|---|---|---|
| **esbuild** | bundle + transpile | — |
| **tsx** | run `.ts` directly in dev, with watch | yes |
| **tsc** | type-check + emit `.d.ts` | no — it *is* the type checker |
| **tsup / tsdown** | zero-config library builds | tsup yes; tsdown uses Rolldown |

The split to remember: **esbuild transpiles, tsc type-checks.**

### esbuild does not type-check

The one thing to internalise. esbuild strips types *blind* — it won't catch a type error. So you run both, separately:

```bash
tsc --noEmit                    # type-check — the real error gate (CI)
esbuild src/index.ts --bundle   # build — fast, no checking
```

---

## Core usage

```bash
npm i -D esbuild
```

```bash
esbuild src/index.ts \
  --bundle \                  # follow imports into one file (OFF by default)
  --platform=node \           # node | browser | neutral
  --target=node20 \           # syntax floor
  --format=esm \              # esm | cjs | iife
  --outfile=dist/index.js \
  --sourcemap \
  --minify
```

Key flags:

- `--bundle` — merge imports into the output. Without it, esbuild just transpiles each file.
- `--platform=node` — resolve for Node, mark built-ins external, default to CJS.
- `--packages=external` — don't bundle anything from `node_modules` (servers & libraries).
- `--external:pkg` — exclude one dependency (e.g. native addons, the AWS SDK).
- `--format` — `esm` / `cjs` / `iife`.
- `--target` — `node20`, `es2022`; the lowest syntax you support.
- `--watch` — rebuild on change (but prefer tsx for dev — below).

JS API, for build scripts:

```ts
import { build } from "esbuild";

await build({
  entryPoints: ["src/index.ts"],
  bundle: true,
  platform: "node",
  target: "node20",
  format: "esm",
  outfile: "dist/index.js",
  packages: "external",       // keep node_modules out of the bundle
});
```

---

## Common scenarios

### Bundle a Lambda function

A small single file = fast cold start. Externalize the AWS SDK (the runtime already provides it):

```bash
esbuild src/handler.ts --bundle --platform=node --target=node20 \
  --format=esm --outfile=dist/handler.mjs --external:@aws-sdk/*
```

> See [aws-lambda.md](aws-lambda.md). CDK's `NodejsFunction` and SST run esbuild for you under the hood.

### Run TS in dev (use tsx, not esbuild directly)

```bash
npx tsx watch src/server.ts     # run + reload on change, no build step (esbuild inside)
```

Or Node's native type stripping (Node 22+):

```bash
node --experimental-strip-types src/server.ts
```

### Type-check (tsc — esbuild won't)

```bash
tsc --noEmit                    # the only thing that actually catches type errors
```

### Build a library (use tsup, not raw esbuild)

tsup wraps esbuild and adds dual format + types in one step:

```bash
npx tsup src/index.ts --format esm,cjs --dts --clean
```

Then point `package.json` at the outputs and **externalize deps** (don't ship copies):

```jsonc
{
  "type": "module",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.js",
      "require": "./dist/index.cjs"
    }
  }
}
```

---

## Practical recipes

**Bundle a Node service to one file, deps left external:**
```bash
esbuild src/index.ts --bundle --platform=node --packages=external --outfile=dist/index.js
```

**Bundle for Lambda (ESM, SDK external):**
```bash
esbuild src/handler.ts --bundle --platform=node --format=esm --target=node20 \
  --external:@aws-sdk/* --outfile=dist/handler.mjs
```

**Dev with reload (tsx):**
```bash
npx tsx watch src/server.ts
```

**Type-check gate in CI:**
```bash
tsc --noEmit
```

**One-off transpile, no bundle:**
```bash
esbuild src/index.ts --outfile=dist/index.js
```

**Build a dual ESM/CJS library with types:**
```bash
npx tsup src/index.ts --format esm,cjs --dts --clean
```

---

## Failure modes & fixes

| Symptom | Cause | Fix |
|---|---|---|
| Type errors slip into prod | esbuild doesn't type-check | run `tsc --noEmit` in CI |
| Bundle huge / `node_modules` inlined | bundling deps you didn't need to | `--packages=external` (servers & libs) |
| `__dirname is not defined` | ESM output | derive from `import.meta.url` (see [nodejs.md](nodejs.md)) |
| Native/binary dep crashes when bundled | native addons can't be bundled | mark it `--external:<pkg>` |
| Lambda artifact bloated | bundling the AWS SDK | `--external:@aws-sdk/*` |
| Library ships a duplicate of a peer dep | bundled instead of externalized | externalize peer deps |

## Tips

- esbuild transpiles; tsc type-checks — always run **both**, they catch different things.
- Servers and libraries → `--packages=external`. Only inline deps when you need a self-contained artifact (Lambda, Workers).
- Use **tsx** for dev, not a hand-rolled esbuild `--watch`.
- For libraries, reach for **tsup / tsdown** — don't hand-roll dual ESM/CJS + types over raw esbuild.
- Set `--target` to the lowest runtime you actually support, no lower.
- Decide *whether* to bundle by deploy target first — a plain server often shouldn't.
