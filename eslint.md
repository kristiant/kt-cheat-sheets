# ESLint

**What it is:** A pluggable **linter** for JavaScript/TypeScript — it statically analyses code against configurable rules and reports (or auto-fixes) problems.

**Why people use it:** Catches bugs and bad patterns before runtime, and enforces a consistent style across a team automatically, in the editor and in CI.

**Typically used for:** Enforcing code quality/standards on JS/TS projects, catching common mistakes (unused vars, bad `await`, accidental `any`), and gating PRs in CI.

> Modern ESLint (v9+) uses **flat config** (`eslint.config.js`). The old `.eslintrc.*` format is legacy. This sheet uses flat config.

---

## Setup

```bash
npm init @eslint/config@latest      # interactive scaffold (recommended)
# or manually:
npm i -D eslint
```

## Most common commands

```bash
npx eslint .                        # lint everything
npx eslint src/                     # lint a directory
npx eslint . --fix                  # auto-fix what it can
npx eslint . --max-warnings 0       # fail if ANY warnings (good for CI)
npx eslint app.ts --debug           # see why config resolves as it does
npx eslint . --no-cache             # ignore cache
npx eslint . --cache                # cache results (faster re-runs)
```

## Simple usage — flat config

```js
// eslint.config.js
import js from "@eslint/js";

export default [
  js.configs.recommended,
  {
    rules: {
      "no-unused-vars": "warn",
      "no-console": "off"
    }
  }
];
```

```bash
npx eslint .          # lint
npx eslint . --fix    # fix
```

Rule severities: `"off"` / `"warn"` / `"error"` (or `0` / `1` / `2`).

---

## TypeScript setup (typescript-eslint)

The standard way to lint TS:

```bash
npm i -D eslint @eslint/js typescript typescript-eslint
```

```js
// eslint.config.js
import js from "@eslint/js";
import tseslint from "typescript-eslint";

export default tseslint.config(
  js.configs.recommended,
  ...tseslint.configs.recommended,        // or recommendedTypeChecked
  {
    rules: {
      "@typescript-eslint/no-unused-vars": "error",
      "@typescript-eslint/no-explicit-any": "warn"
    }
  }
);
```

> **Grounded:** [typescript-eslint](https://typescript-eslint.io/) is the official toolchain for linting TS and is what virtually every TS project uses. Their `recommendedTypeChecked` preset enables rules that use the type-checker (e.g. catching floating promises) — more powerful but requires pointing ESLint at your `tsconfig`.

---

## Advanced

### Type-aware linting (catches floating promises, unsafe `any`)

```js
import tseslint from "typescript-eslint";

export default tseslint.config(
  ...tseslint.configs.recommendedTypeChecked,
  {
    languageOptions: {
      parserOptions: {
        projectService: true,            // auto-find tsconfig
        tsconfigRootDir: import.meta.dirname
      }
    }
  }
);
```

### Ignoring files (flat config has no `.eslintignore`)

```js
export default [
  { ignores: ["dist/", "build/", "coverage/", "**/*.config.js"] },
  // ...rest of config
];
```

### Per-area overrides (different rules for tests vs src)

```js
export default [
  js.configs.recommended,
  {
    files: ["**/*.test.ts"],
    rules: { "@typescript-eslint/no-explicit-any": "off" }
  },
  {
    files: ["src/**/*.ts"],
    rules: { "no-console": "error" }
  }
];
```

### Pair with Prettier (formatting vs linting)

Let Prettier own formatting; turn off ESLint's stylistic rules that would conflict:

```bash
npm i -D eslint-config-prettier
```

```js
import prettier from "eslint-config-prettier";
export default [ js.configs.recommended, prettier ]; // prettier LAST to win
```

> **Grounded:** The modern consensus (per [typescript-eslint](https://typescript-eslint.io/users/configs) and Prettier's own docs) is to use ESLint for *code-quality* rules and Prettier for *formatting*, with `eslint-config-prettier` disabling the overlap — rather than ESLint's deprecated formatting rules.

### Inline disabling (use sparingly)

```ts
// eslint-disable-next-line @typescript-eslint/no-explicit-any
const x: any = legacyApi();

/* eslint-disable no-console */   // disable for a block
console.log("debug");
/* eslint-enable no-console */
```

---

## Practical recipes

**The standard CI lint gate (no warnings allowed):**
```bash
npx eslint . --max-warnings 0
```

**Auto-fix everything fixable across the repo:**
```bash
npx eslint . --fix
```

**Lint only files changed vs main (fast pre-commit):**
```bash
npx eslint $(git diff --name-only --diff-filter=ACM main -- '*.ts' '*.tsx')
```

**Speed up repeated runs with caching:**
```bash
npx eslint . --cache --cache-location node_modules/.cache/eslint/
```

**Find which config/rule applies to one file (debug surprises):**
```bash
npx eslint --print-config src/app.ts
```

**Output JSON for tooling / CI annotations:**
```bash
npx eslint . -f json -o eslint-report.json
```

**Run only a single rule across the codebase (audit one thing):**
```bash
npx eslint . --rule '{"no-console": "error"}' --no-eslintrc
```

**add a package.json script (so the team runs it the same way):**
```json
{ "scripts": { "lint": "eslint .", "lint:fix": "eslint . --fix" } }
```

---

## Tips

- Use **flat config** (`eslint.config.js`) on ESLint v9+. Don't start new projects on `.eslintrc`.
- `--max-warnings 0` in CI turns warnings into hard failures — useful so warnings don't accumulate.
- Let **Prettier format, ESLint lint**; wire `eslint-config-prettier` last to kill conflicts.
- Type-aware rules (`recommendedTypeChecked`) catch real bugs (floating promises) but need a `tsconfig` and are slower.
- Order matters in flat config — later entries override earlier ones.
- Run ESLint and `tsc --noEmit` as separate CI steps; they catch different things.
