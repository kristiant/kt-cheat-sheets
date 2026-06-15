# Bundlers (Vite & friends)

**What it is:** Build tools that take your source modules and produce optimised output — transpiling, bundling, tree-shaking, and code-splitting for the browser or Node.

**Why people use it:** Browsers and npm need processed output, not raw TS with bare imports. Bundlers also give a fast dev server with hot reload, and shrink/optimise production builds.

**Typically used for:** Building web apps (Vite), bundling libraries for publishing (tsup/Rolldown), and fast transpilation (esbuild).

> For how imports *resolve* to files, see [module-resolution.md](module-resolution.md).

---

## The toolchain (three layers people conflate)

| Layer | Job | Tools |
|---|---|---|
| **Transpiler** | TS/JSX → JS, downlevel syntax. One file at a time. | esbuild, swc, Oxc, babel, tsc |
| **Bundler** | Merge modules into output, tree-shake, split | Rolldown, Rollup, esbuild, webpack |
| **Dev server / orchestrator** | HMR, config, plugins over the above | Vite |

Vite is not itself a bundler — it *orchestrates* a transpiler + bundler and adds a dev server.

---

## Vite

The default for web apps. `npm create vite@latest`.

```ts
// vite.config.ts
import { defineConfig } from "vite";
export default defineConfig({
  resolve: { alias: { "@": "/src" } },
  server: { port: 3000 },
  build: { sourcemap: true, target: "es2022" },
});
```

```bash
vite          # dev server with HMR
vite build    # production build → dist/
vite preview  # serve the built output locally
```

### Dev vs build

- **Dev** — serves source over native ESM, transforms on demand. Near-instant startup; only the modules you load are processed.
- **Build** — bundles, tree-shakes, minifies, code-splits into `dist/`.

> **Vite 8 (2026)** unified both on **Rolldown** (Rust). Earlier Vite used esbuild for dev + Rollup for build — that split caused occasional "works in dev, breaks in build" bugs, now largely gone with one bundler across both. On Vite ≤7 the asymmetry still applies.

### Env vars & globs

```ts
import.meta.env.VITE_API_URL;          // only VITE_-prefixed vars are exposed to client
import.meta.env.DEV;                   // true in dev
const mods = import.meta.glob("./pages/*.ts");   // dynamic import map (lazy by default)
const eager = import.meta.glob("./icons/*.svg", { eager: true });
```

### Dynamic import (code-splitting)

```ts
const { heavy } = await import("./heavy.js");   // split into its own chunk, loaded on demand
```

---

## Library bundling

Apps use Vite; **libraries** have different needs — emit multiple formats, externalize deps, ship types. Use **tsup** (esbuild-based, the safe default) or **tsdown** (Rolldown-based successor, faster).

```bash
npm i -D tsup
```

```ts
// tsup.config.ts
import { defineConfig } from "tsup";
export default defineConfig({
  entry: ["src/index.ts"],
  format: ["esm", "cjs"],   // dual output
  dts: true,                // emit .d.ts (+ .d.cts) types
  sourcemap: true,
  clean: true,
  external: ["react"],      // don't bundle peer deps — see below
});
```

Wire the outputs into `package.json` so consumers resolve the right format:

```jsonc
{
  "type": "module",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.js",
      "require": "./dist/index.cjs"
    }
  },
  "files": ["dist"]
}
```

> **Externalize, don't bundle, dependencies in a library.** Bundling `react` into your lib ships a second copy and breaks hooks. Mark deps `external` (peer deps especially); only apps bundle everything.

App vs library, at a glance:

| | App | Library |
|---|---|---|
| Tool | Vite | tsup / tsdown / Rollup |
| Deps | bundled in | externalized |
| Output | one optimised build | ESM + CJS + `.d.ts` |
| Entry | `index.html` | `package.json` `exports` |

---

## Build concepts

**Tree-shaking** — drop unused exports. Needs ESM (static imports) and an honest `sideEffects` flag:

```jsonc
{ "sideEffects": false }   // or ["*.css"] — tells the bundler nothing breaks if unused code is dropped
```

**Code-splitting** — `import()` and route boundaries become separate chunks, loaded on demand (smaller initial payload).

**Minify / target** — shrink output; `target` sets the syntax floor (`es2022`, `node20`). Lower target = more downleveling = bigger output.

**Externals** — exclude from the bundle (Node built-ins, peer deps for libs).

**Source maps** — map built code back to source for debugging; ship them or keep them private.

**`define` / env** — inline constants at build time:

```ts
// vite.config.ts → define: { __VERSION__: JSON.stringify("1.2.3") }
```

**Bundle analysis** — find what's bloating output:

```bash
npx vite-bundle-visualizer        # treemap of chunk sizes
```

---

## The other tools (what + when)

| Tool | What | Reach for it when |
|---|---|---|
| **esbuild** | Go-based transpiler+bundler, very fast | quick transpile, tooling internals |
| **Rollup** | Mature ESM bundler, great tree-shaking | libraries (pre-Rolldown), plugins |
| **Rolldown** | Rust, Rollup-compatible; powers Vite 8 | the modern Rollup successor |
| **tsup / tsdown** | Zero-config library bundlers | publishing a package |
| **webpack** | Old, powerful, sprawling config | legacy apps, niche loaders |
| **Bun** | Runtime with a built-in fast bundler | all-in-one Bun projects |

---

## Practical recipes

**Scaffold a Vite + TS app:**
```bash
npm create vite@latest my-app -- --template vanilla-ts
```

**Bundle a dual ESM/CJS library with types:**
```bash
npx tsup src/index.ts --format esm,cjs --dts --clean
```

**Analyse what's in the production bundle:**
```bash
npx vite-bundle-visualizer
```

**Lazy-load a heavy module:**
```ts
const mod = await import("./heavy.js");   // own chunk, fetched on first use
```

**Expose an env var to the client (Vite):**
```bash
# .env →  VITE_API_URL=https://api.example.com   (must be VITE_-prefixed)
```

**Externalize a peer dep in a library build (tsup):**
```ts
export default defineConfig({ external: ["react", "react-dom"] });
```

---

## Failure modes & fixes

| Symptom | Cause | Fix |
|---|---|---|
| Env var is `undefined` in the browser | missing `VITE_` prefix | prefix it; only `VITE_*` is exposed client-side |
| Library ships a duplicate React | bundled instead of externalized | mark peer deps `external` |
| Huge initial bundle | no code-splitting | `import()` at route/feature boundaries |
| Dead code not removed | `sideEffects` not set / CJS imports | `"sideEffects": false`, use ESM imports |
| Types missing for consumers | no `.d.ts` emitted / not in `exports` | `dts: true` + `types` condition in `exports` |
| "works in dev, breaks in build" (Vite ≤7) | esbuild-dev / Rollup-build divergence | align config; upgrade to Vite 8 (single Rolldown) |

## Tips

- Apps bundle dependencies; libraries externalize them — getting this backwards is the most common lib bug.
- Tree-shaking needs ESM and a correct `sideEffects` field; CJS and side-effectful imports defeat it.
- Only `VITE_`-prefixed env vars reach client code — never expose secrets this way.
- Ship dual ESM/CJS only if you must support CJS consumers; ESM-only is simpler and increasingly fine.
- `target` trades compatibility for size — set it to the lowest runtime you actually support, not lower.
- On Vite 8+ the dev/prod bundler is unified (Rolldown); the old "works in dev only" class of bug is mostly gone.
