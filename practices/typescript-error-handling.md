# TypeScript Error Handling

**What it is:** A convention for designing errors as a typed hierarchy — a base error carrying metadata, a small taxonomy by *cause*, narrow domain subclasses, and a parallel HTTP error tree.

**Why people use it:** It makes errors carry their own severity and reporting policy, lets you decide "is this our bug or the user's?" by type alone, and keeps message-building out of call sites.

**Typically used for:** Backends that log/report to Sentry, distinguish recoverable from fatal failures, and map internal errors to HTTP responses.

> Grounded in [n8n](https://github.com/n8n-io/n8n) (`packages/workflow/src/errors`, `packages/cli/src/errors`).

---

## Classify by cause, not by location

Three base classes, each with a sensible default log level. Pick by asking "whose fault is it?"

| Class | Meaning | Whose fault | Default level | Report to Sentry? |
|---|---|---|---|---|
| `UserError` | Bad input, unauthorized, business-rule violation | **User** | `info` | no |
| `OperationalError` | Transient: network blip, DB timeout, expected to happen | **Environment** | `warning` | no |
| `UnexpectedError` | Logic bug, unhandled case, failed assertion | **Developer** | `error` | yes |

This single decision drives logging, alerting, and whether you wake someone up.

```ts
import { UserError, OperationalError, UnexpectedError } from 'n8n-workflow';

if (!email.includes('@')) throw new UserError('Invalid email address');
if (!dbResponse)          throw new OperationalError('Database query timed out');
if (state === undefined)  throw new UnexpectedError('Unreachable: state must be set');
```

---

## The base error — metadata, not just a message

A shared `BaseError extends Error` carries reporting metadata so handlers don't have to guess.

```ts
export abstract class BaseError extends Error {
  level: ErrorLevel;              // 'fatal' | 'error' | 'warning' | 'info'
  readonly shouldReport: boolean; // send to Sentry? defaults from level
  readonly description?: string | null;
  readonly tags: ErrorTags;       // structured context for the reporter
  readonly extra?: Record<string, unknown>;

  constructor(message: string, { level = 'error', description, shouldReport, tags = {}, extra, ...rest }: BaseErrorOptions = {}) {
    super(message, rest);                              // `rest` carries { cause }
    this.level = level;
    this.shouldReport = shouldReport ?? (level === 'error' || level === 'fatal');
    this.description = description;
    this.tags = tags;
    this.extra = extra;
  }
}
```

The three taxonomy classes just set the default `level`:

```ts
export class UserError extends BaseError {        // default level: info, not reported
  constructor(message: string, opts: UserErrorOptions = {}) {
    super(message, { ...opts, level: opts.level ?? 'info' });
  }
}
```

**Preserve the cause** — always chain the original error so the stack survives:

```ts
try { await fetchRemote(); }
catch (e) { throw new OperationalError('Failed to sync', { cause: e }); }
```

---

## Domain errors — subclass with a message-building constructor

Don't sprinkle string literals at throw sites. Give each domain error a class whose constructor builds the message from inputs. Throwing becomes `throw new CredentialNotFoundError(id)`.

```ts
import { UserError } from 'n8n-workflow';

export class CredentialNotFoundError extends UserError {
  constructor(credentialId: string, credentialType?: string) {
    super(
      credentialType
        ? `Credential with ID "${credentialId}" does not exist for type "${credentialType}".`
        : `Credential with ID "${credentialId}" was not found.`,
    );
  }
}
```

Benefits: consistent messages, greppable (`grep -r CredentialNotFoundError`), and `instanceof`-checkable. One error type per `*.error.ts` file.

---

## HTTP layer — a parallel `ResponseError` tree

Keep transport concerns out of business errors. Services throw *domain/`UserError`*; the HTTP layer has its own tree carrying status codes, and an error handler maps between them.

```ts
import { ResponseError } from './abstract/response.error';

export class NotFoundError extends ResponseError {
  constructor(message: string, hint?: string) {
    super(message, 404, 404, hint);   // (message, httpStatus, errorCode, hint)
  }
}
// siblings: BadRequestError (400), ForbiddenError (403), ConflictError (409), …
```

A global error middleware then does the mapping in one place:

```ts
if (error instanceof ResponseError) return res.status(error.httpStatusCode).json({ message: error.message });
if (error instanceof UserError)     return res.status(400).json({ message: error.message });
// otherwise: log + report (it's an UnexpectedError / unknown) and return a generic 500
```

---

## Assertion helpers — throw-on-null with narrowing

Attach static assertions to the error so a guard both throws *and* narrows the type for the compiler.

```ts
export class NotFoundError extends ResponseError {
  static isDefinedAndNotNull<T>(
    value: T | undefined | null,
    message: string,
  ): asserts value is T {
    if (value === undefined || value === null) throw new NotFoundError(message);
  }
}

// usage — after this line, `workflow` is narrowed to non-null
const workflow = await repo.findById(id);
NotFoundError.isDefinedAndNotNull(workflow, `Workflow ${id} not found`);
workflow.name;   // ✅ no `!`, no optional chaining
```

---

## Normalizing unknown caught values

In a `catch`, the value is `unknown`. Normalize to an `Error` before using it.

```ts
function ensureError(e: unknown): Error {
  return e instanceof Error ? e : new Error(String(e));
}

try { … } catch (e) {
  const err = ensureError(e);
  logger.error(err.message, { stack: err.stack });
  throw new UnexpectedError('Processing failed', { cause: err });
}
```

---

## Rules of thumb

- **Deprecate the catch-all.** A generic `ApplicationError` invites lazy throws — mark it `@deprecated` and push people to the taxonomy.
- **Never throw a bare string or plain `Error`** in app code — you lose level, reporting, and `instanceof` checks.
- **Choose the class by cause**, then override `level` only when this specific case differs from the default.
- **`UserError` ≠ alert.** If it's the user's fault, don't page yourself — that's what `shouldReport: false` (via `info`) encodes.

> See also: [typescript-backend-architecture.md](typescript-backend-architecture.md), [typescript-patterns.md](../cheatsheets/typescript-patterns.md) for `asserts`/type guards.
