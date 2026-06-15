# JSDoc

**What it is:** A comment syntax (`/** ... */`) for documenting JavaScript — and, with TypeScript's tooling, for *typing* plain JS without writing TS.

**Why people use it:** Editor hover docs and autocomplete, type-checking JS via `// @ts-check` (no build step), and shipping types alongside JS libraries.

**Typically used for:** Documenting function params/returns, type-checking JS codebases that can't adopt `.ts`, and annotating public APIs.

> TypeScript understands JSDoc, so most type features have a JSDoc equivalent. Reference: [TS JSDoc docs](https://www.typescriptlang.org/docs/handbook/jsdoc-supported-types.html).

---

## Anatomy

A JSDoc block is a `/** */` comment directly above the thing it describes:

```js
/**
 * Charge a customer's card.
 * @param {string} customerId - Stripe customer id
 * @param {number} amountCents - amount in cents
 * @returns {Promise<string>} the charge id
 */
async function charge(customerId, amountCents) { /* ... */ }
```

Editors surface the description, params, and return type on hover and in autocomplete.

---

## Common documentation tags

```js
/**
 * @param {string} name        - a described parameter
 * @param {number} [age]       - optional parameter (square brackets)
 * @param {string} [role="user"] - optional with a default
 * @returns {boolean}
 * @throws {RangeError} when amount is negative
 * @example
 *   greet("Ada");  // "Hello, Ada"
 * @deprecated use `greetUser` instead
 * @see https://example.com/docs
 */
```

Destructured / nested params use dotted names:

```js
/**
 * @param {object} opts
 * @param {string} opts.host
 * @param {number} [opts.port=5432]
 */
function connect({ host, port }) {}
```

---

## Typing JS with JSDoc

Enable checking per-file with `// @ts-check`, or project-wide in `tsconfig`:

```jsonc
{ "compilerOptions": { "checkJs": true, "allowJs": true, "noEmit": true } }
```

```js
// @ts-check

/** @type {string} */
let title;

/** @type {{ id: number, name: string }} */
const user = { id: 1, name: "Ada" };   // checked — missing/extra keys error

const score = /** @type {number} */ (raw);   // inline cast (note the parens)
```

### Reusable types — `@typedef`

```js
/**
 * @typedef {object} User
 * @property {number} id
 * @property {string} name
 * @property {string} [email]   - optional property
 */

/** @param {User} user */
function save(user) {}
```

Inline object form for short types:

```js
/** @typedef {{ id: number, name: string }} User */
```

### Function types — `@callback`

```js
/**
 * @callback Comparator
 * @param {number} a
 * @param {number} b
 * @returns {number}
 */

/** @param {Comparator} cmp */
function sort(cmp) {}
```

### Import types from elsewhere

```js
/** @import { User } from "./types.js" */   // TS 5.5+ import-style
/** @param {User} u */
function f(u) {}

// or inline, anywhere a type is expected:
/** @type {import("./types.js").User} */
let u;
```

---

## Type expressions

```js
/** @type {string | null} */          // union
/** @type {string=} */                // optional (= suffix)
/** @type {?number} */                // nullable (number | null)
/** @type {string[]} */               // array  (or Array<string>)
/** @type {Record<string, number>} */ // map
/** @type {[string, number]} */       // tuple
/** @type {(a: number) => boolean} */ // function
/** @type {Promise<User>} */          // generic
/** @type {"a" | "b" | "c"} */        // string-literal union
/** @type {*} */                      // any
/** @type {unknown} */
```

---

## Advanced

### Generics — `@template`

```js
/**
 * @template T
 * @param {T[]} arr
 * @returns {T | undefined}
 */
function first(arr) { return arr[0]; }
```

Constrain a generic:

```js
/**
 * @template {string | number} K
 * @param {K} key
 */
function use(key) {}
```

### `@satisfies` — validate without widening

The JSDoc form of TypeScript's `satisfies` (TS 5.1+):

```js
/** @satisfies {Record<string, number>} */
const config = { a: 1, b: 2 };   // checked against the type, but keeps its literal shape
```

### `@overload`

```js
/**
 * @overload
 * @param {string} x
 * @returns {string}
 *//**
 * @overload
 * @param {number} x
 * @returns {number}
 */
/** @param {string | number} x */
function id(x) { return x; }
```

### Enums and access

```js
/** @enum {string} */
const Status = { Open: "open", Closed: "closed" };

class Repo {
  /** @private */
  #cache = new Map();
  /** @readonly */
  name = "repo";
}
```

> **Grounded:** TypeScript ships first-class JSDoc support precisely so JS projects get type-checking without `.ts` — the [TS handbook's JSDoc reference](https://www.typescriptlang.org/docs/handbook/jsdoc-supported-types.html) is the source of truth for which tags affect types. Projects like SvelteKit have shipped large JSDoc-typed JS codebases on this basis.

---

## Practical recipes

**Type-check a single JS file (no config):**
```js
// @ts-check   ← first line of the file
```

**Type-check a whole JS project:**
```jsonc
// tsconfig.json
{ "compilerOptions": { "allowJs": true, "checkJs": true, "noEmit": true }, "include": ["src"] }
```

**Generate `.d.ts` types from JSDoc-annotated JS (ship types from a JS lib):**
```bash
tsc src/index.js --declaration --emitDeclarationOnly --allowJs --outDir dist
```

**Cast a value inline (note the required parentheses):**
```js
const el = /** @type {HTMLInputElement} */ (document.getElementById("name"));
```

**Silence one false positive:**
```js
// @ts-expect-error - third-party types are wrong here
doThing(weirdArg);
```

**Reuse a type across files:**
```js
/** @typedef {import("./user.js").User} User */
```

---

## Tips

- Add `// @ts-check` to a JS file to get TS-level checking with zero build setup.
- Optional params: `[name]` in the description, or `string=` / `?` in the type — pick one style and keep it.
- Prefer `@typedef` (or a real `.ts` types file imported via `import("...")`) over repeating inline object types.
- Inline casts need parentheses: `/** @type {T} */ (value)` — a common gotcha.
- For libraries, write JSDoc on JS and emit `.d.ts` with `tsc` — consumers get types, you skip a build.
- If you're already on TypeScript, use real TS syntax; JSDoc types are for JS that can't be `.ts`.
