# Module Resolution (JS/TS)

**What it is:** The rules that map an `import "x"` to an actual file on disk — Node's algorithm, `package.json` fields, and the TypeScript/bundler resolvers layered on top.

**Why people use it:** Most "cannot find module", "require of ES Module", and "works in dev but not prod" errors are resolution problems. Knowing the rules makes them fixable instead of mysterious.

**Typically used for:** Configuring TS projects, publishing packages, setting up path aliases, and debugging import failures.

---

## Mental model

Resolution answers one question: **given a specifier, which file loads?** Three resolvers must agree, or you get errors:

| Resolver | Reads | Used when |
|---|---|---|
| **Node** (runtime) | `package.json` `exports`/`main`, file extensions | the code actually runs |
| **TypeScript** | `tsconfig` `moduleResolution`, `paths`, `types` | type-checking |
| **Bundler** (Vite/etc.) | its own config `alias` + `package.json` | building |

A path alias that works in TS but crashes at runtime = the resolvers disagree. See [Path aliases](#path-aliases).

Specifier kinds:

```ts
import x from "./util.js";   // relative — a path
import y from "lodash";      // bare — resolved via node_modules + exports
import z from "#config";     // subpath import — resolved via package.json "imports"
```

---

## ESM vs CJS

```js
import { x } from "./m.js";   // ESM — static, async, tree-shakeable
const { x } = require("./m"); // CJS — dynamic, synchronous
```

`package.json` `"type"` decides how `.js` is interpreted: `"module"` → ESM, `"commonjs"` (or absent) → CJS. Force per-file with `.mjs` / `.cjs`. See [nodejs.md](nodejs.md).

---

## package.json resolution fields

```jsonc
{
  "type": "module",            // how .js files are parsed (module | commonjs)
  "main": "./dist/index.cjs",  // legacy entry (CJS fallback, old tools)
  "module": "./dist/index.js", // legacy ESM entry (bundler convention, unofficial)
  "types": "./dist/index.d.ts",// TypeScript types entry
  "exports": { /* ... */ },    // modern entry map — overrides main/module
  "imports": { /* ... */ }     // internal #subpath aliases
}
```

`exports` **takes precedence** over `main`/`module` in modern Node and bundlers, and it *encapsulates* the package — consumers can only import the subpaths you list.

---

## The `exports` map

Conditional exports pick a target by how the file is imported. **Order matters** — `types` first, `default` last; Node uses the first match:

```jsonc
{
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.js",      // ESM consumers (import)
      "require": "./dist/index.cjs",    // CJS consumers (require)
      "default": "./dist/index.js"      // fallback
    },
    "./utils": {                        // subpath export: import "my-lib/utils"
      "types": "./dist/utils.d.ts",
      "import": "./dist/utils.js"
    },
    "./package.json": "./package.json"  // explicitly expose if needed
  }
}
```

Common conditions: `types`, `import`, `require`, `browser`, `node`, `development`, `production`, `default`.

> Listing `exports` **blocks deep imports** you didn't declare — `import "my-lib/dist/secret"` fails unless mapped. That's the point (a real public API), but it surprises people.

### `imports` — internal aliases

Package-private subpaths (must start with `#`), resolved without a bundler:

```jsonc
{ "imports": { "#config": "./src/config.js", "#*": "./src/*.js" } }
```

```ts
import { db } from "#config";   // resolves inside this package only
```

---

## tsconfig `moduleResolution`

Tells TypeScript *how* to resolve. The two that matter today:

```jsonc
{
  "compilerOptions": {
    "moduleResolution": "bundler",  // for Vite/esbuild/bundled apps — understands exports, no extensions needed
    // "moduleResolution": "nodenext" // for code that runs directly in Node — enforces extensions, exports conditions
  }
}
```

- **`bundler`** — assume a bundler resolves imports; relaxed (no file extensions in imports). Default for Vite apps.
- **`nodenext`** — match Node's real ESM rules; requires `.js` extensions in imports, respects `type`/`exports`. Use for published Node libraries.

See [typescript.md](typescript.md). Legacy `node`/`node10` only for old CJS projects.

---

## Path aliases

`@/* → src/*` so you write `import "@/lib/db"` not `../../lib/db`. The catch: **every resolver needs to know about it.**

```jsonc
// tsconfig.json — makes TYPE-CHECKING resolve the alias
{ "compilerOptions": { "baseUrl": ".", "paths": { "@/*": ["src/*"] } } }
```

```ts
// vite.config.ts — makes the BUILD/dev server resolve it
export default defineConfig({
  resolve: { alias: { "@": "/src" } },
});
```

> `tsc` understands `paths` for type-checking but **does not rewrite them on emit** — plain `tsc` output crashes at runtime. A bundler (Vite), `tsc-alias`, or a runtime loader (`tsconfig-paths`) must do the rewrite. This is the #1 alias gotcha.

---

## Practical recipes

**Dual ESM/CJS package entry (for publishing):**
```jsonc
"exports": {
  ".": {
    "types": "./dist/index.d.ts",
    "import": "./dist/index.js",
    "require": "./dist/index.cjs"
  }
}
```

**Match Node's real resolution in a library:**
```jsonc
{ "compilerOptions": { "module": "nodenext", "moduleResolution": "nodenext" } }
```

**Add a path alias that works everywhere (TS app):**
```jsonc
// tsconfig: "paths": { "@/*": ["src/*"] }
// vite.config: resolve.alias { "@": "/src" }
```

**Expose only a public API (block deep imports):**
```jsonc
"exports": { ".": "./dist/index.js" }   // consumers can't reach ./dist/internal
```

**Get `__dirname` in ESM (it doesn't exist):**
```ts
import { fileURLToPath } from "node:url";
const __dirname = fileURLToPath(new URL(".", import.meta.url));
```

---

## Failure modes & fixes

| Error / symptom | Cause | Fix |
|---|---|---|
| `require() of ES Module not supported` | CJS file `require`-ing an ESM-only package | use dynamic `import()`, or set `"type": "module"` |
| `__dirname is not defined` | `__dirname`/`require` don't exist in ESM | derive from `import.meta.url` |
| `Cannot find module '@/x'` at runtime | alias resolved by TS but not rewritten on emit | bundler / `tsc-alias` / `tsconfig-paths` |
| `Package subpath './x' is not defined by exports` | importing a path not in the `exports` map | add the subpath, or import a declared one |
| Types resolve but import fails (or vice versa) | `exports` missing a `types`/`import`/`require` condition | add the missing condition (and `types` first) |
| Dual-package hazard (two copies of state) | a dep loaded as both ESM and CJS | single format, or share state via a CJS core |

## Tips

- `exports` overrides `main`/`module` and encapsulates the package — list every public subpath, including `./package.json` if tools need it.
- In `exports`, put `types` first and `default` last; Node takes the first matching condition.
- `moduleResolution: bundler` for bundled apps, `nodenext` for code Node runs directly.
- Aliases must be mirrored in *both* tsconfig `paths` and the bundler — and rewritten at build/runtime, since `tsc` won't.
- Prefer ESM for new packages; ship CJS only via a `require` condition if you must support old consumers.
