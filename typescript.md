# TypeScript (& tsconfig)

**What it is:** A typed superset of JavaScript that compiles to plain JS. You write JS plus type annotations; the compiler (`tsc`) checks them and strips them out.

**Why people use it:** Catches whole classes of bugs at author-time (typos, wrong shapes, null misuse), gives editors real autocomplete/refactoring, and serves as always-correct documentation.

**Typically used for:** Any non-trivial JS codebase — frontend (React/Vue), Node backends, libraries, and CLIs.

---

## Setup

```bash
npm i -D typescript
npx tsc --init           # generate a tsconfig.json
npx tsc                  # type-check + compile
npx tsc --noEmit         # type-check only (no JS output) — common in CI
npx tsc --watch          # recompile on change
```

## Most used `tsconfig.json` options

```jsonc
{
  "compilerOptions": {
    "target": "ES2022",          // JS version to emit
    "module": "NodeNext",        // module system (NodeNext for modern Node)
    "moduleResolution": "NodeNext",
    "outDir": "./dist",          // where compiled JS goes
    "rootDir": "./src",          // source root
    "strict": true,              // enable all strict checks
    "esModuleInterop": true,     // clean default imports of CJS modules
    "skipLibCheck": true,        // don't type-check .d.ts files (faster)
    "declaration": true,         // emit .d.ts files (for libraries)
    "sourceMap": true,           // debugging support
    "resolveJsonModule": true,   // import .json files
    "noEmit": false              // set true when another tool does the emit
  },
  "include": ["src"],
  "exclude": ["node_modules", "dist"]
}
```

---

## A sensible starting tsconfig

### Modern Node app

```jsonc
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "outDir": "dist",
    "rootDir": "src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "sourceMap": true
  },
  "include": ["src"]
}
```

### Bundler-based frontend (Vite/Next/esbuild does the emit)

```jsonc
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "jsx": "react-jsx",
    "strict": true,
    "skipLibCheck": true,
    "noEmit": true,              // bundler emits; tsc only type-checks
    "isolatedModules": true      // each file compiles independently
  },
  "include": ["src"]
}
```

---

## Advanced

### What `"strict": true` actually turns on

It's a bundle. The high-value ones:

```jsonc
{
  "strictNullChecks": true,        // null/undefined must be handled explicitly
  "noImplicitAny": true,           // no silent `any`
  "strictFunctionTypes": true,
  "noUncheckedIndexedAccess": true // arr[i] is T | undefined (opt-in)
}
```

> **Grounded:** Strict mode is the community default — the TS team's own `tsc --init` enables `strict`, and frameworks ship it on: [Next.js](https://nextjs.org/docs/app/api-reference/config/typescript) generates `"strict": true`, and the widely-used [`@tsconfig/strictest`](https://github.com/tsconfig/bases) base layers on `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`, etc.

### Extend a shared base config

Don't hand-roll; extend a vetted base:

```jsonc
{
  "extends": "@tsconfig/node20/tsconfig.json",
  "compilerOptions": { "outDir": "dist" },
  "include": ["src"]
}
```

```bash
npm i -D @tsconfig/node20      # or @tsconfig/strictest, @tsconfig/next, ...
```

### Path aliases (clean imports)

```jsonc
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": { "@/*": ["src/*"] }
  }
}
```

```ts
import { db } from "@/lib/db";   // instead of ../../lib/db
```

> Note: `tsc` understands `paths` for *type-checking* but doesn't rewrite them at runtime — your bundler/runtime (Vite, `tsconfig-paths`, Node's loader) must resolve them too.

### Project references (monorepos)

Split a large/monorepo build into independently-compiled, cached units:

```jsonc
{ "references": [{ "path": "../shared" }, { "path": "../api" }] }
```

```bash
tsc --build          # builds refs in dependency order, incrementally
tsc --build --clean  # remove build outputs
```

### Useful type patterns (copy-paste)

```ts
// Derive a type from a value
const config = { port: 3000, env: "dev" } as const;
type Config = typeof config;

// Utility types
type PartialUser = Partial<User>;
type PublicUser  = Omit<User, "passwordHash">;
type UserId      = Pick<User, "id">;
type Role        = "admin" | "user" | "guest";   // union as enum

// Narrowing an unknown (e.g. API response)
function isUser(x: unknown): x is User {
  return typeof x === "object" && x !== null && "id" in x;
}
```

---

## Practical recipes

**Type-check the whole project without emitting (the standard CI gate):**
```bash
npx tsc --noEmit
```

**Watch-mode type-checking while you work:**
```bash
npx tsc --noEmit --watch
```

**Find which file/setting is slowing compiles:**
```bash
npx tsc --noEmit --extendedDiagnostics
```

**Trace why a module won't resolve:**
```bash
npx tsc --traceResolution | grep <module-name>
```

**List every file tsc is actually including (catch stray globs):**
```bash
npx tsc --listFilesOnly
```

**Show the fully-resolved config (after `extends` is merged):**
```bash
npx tsc --showConfig
```

**Build a monorepo with project references, incrementally:**
```bash
npx tsc --build --verbose
```

**Generate only declaration files (`.d.ts`) for a published library:**
```bash
npx tsc --emitDeclarationOnly --declaration --outDir dist/types
```

---

## Tips

- Always start from `"strict": true`. Loosening later is easy; retrofitting strictness onto a loose codebase is painful.
- Use `tsc --noEmit` as a CI step even if a bundler does the actual build — the bundler usually skips type errors.
- Prefer extending a `@tsconfig/*` base over copying options you don't understand.
- `skipLibCheck: true` is near-universal — it skips checking `node_modules` `.d.ts` files and dramatically speeds compiles.
- `paths` aliases need runtime support too; the compiler alone won't rewrite them.
