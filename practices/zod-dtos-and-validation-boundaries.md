# Zod DTOs & Validation Boundaries

**What it is:** A convention for validating untrusted input at the system edge — reusable field schemas, zod-backed DTO *classes*, and a single shared types package across frontend and backend.

**Why people use it:** One source of truth for "what a valid request looks like" — runtime validation and the static type come from the same definition and can't drift, and the same DTO is imported by both client and server.

**Typically used for:** REST/RPC request bodies, query params, and any FE↔BE contract in a TS monorepo.

> Grounded in [n8n](https://github.com/n8n-io/n8n) (`packages/@n8n/api-types`). For Zod syntax itself, see [zod.md](../cheatsheets/zod.md).

---

## The layering: `schemas/` → `dto/`

Split reusable **field schemas** from the **request DTOs** that compose them. Field schemas are shared across many DTOs; DTOs are per-endpoint.

```ts
// schemas/project.schema.ts — reusable, named with `Schema` suffix
import { z } from 'zod';

export const projectNameSchema = z.string().min(1).max(255);
export const projectIconSchema = z.object({
  type: z.enum(['emoji', 'icon']),
  value: z.string().min(1),
});
export type ProjectIcon = z.infer<typeof projectIconSchema>;  // inferred, never hand-written
```

```ts
// dto/project/create-project.dto.ts — composes schemas into a request shape
import { z } from 'zod';
import { projectNameSchema, projectIconSchema } from '../../schemas/project.schema';
import { Z } from '../../zod-class';

export class CreateProjectDto extends Z.class({
  name: projectNameSchema,
  icon: projectIconSchema.optional(),
  uiContext: z.string().optional(),
}) {}
```

---

## Schema-as-class: the `Z.class()` pattern

Instead of a bare schema + a separately-inferred type, wrap the shape in a **class** that carries the schema and validation as statics. The class name *is* the type; no `z.infer` alias needed at the call site.

```ts
// zod-class.ts — the helper (≈40 lines; replaces the `zod-class` npm package)
export const Z = {
  class: <T extends z.ZodRawShape>(shape: T) => {
    const schema = z.object(shape);
    return class {
      static schema = schema;
      static parse(data: unknown) { return schema.parse(data); }       // throws
      static safeParse(data: unknown) { return schema.safeParse(data); } // { success, … }
      static extend<U extends z.ZodRawShape>(extra: U) { return Z.class({ ...shape, ...extra }); }
      constructor(data: z.infer<typeof schema>) { Object.assign(this, schema.parse(data)); }
    } as unknown as ZodClass<z.infer<typeof schema>, T>;
  },
};
```

Why a class rather than a plain schema:
- **One name for value + type** — `CreateProjectDto` is both the validator and the TS type.
- **Reflection-friendly** — a class can be referenced by route decorators (`@Body body: CreateProjectDto`) so the framework knows what to validate.
- **Inheritance** via `.extend()` (below).

---

## DTO inheritance with `.extend()`

Build a specialized DTO from a base without repeating fields:

```ts
export class BaseWorkflowDto extends Z.class({
  name: z.string(),
  nodes: z.array(nodeSchema),
}) {}

export class CreateWorkflowDto extends BaseWorkflowDto.extend({
  projectId: z.string().optional(),
}) {}
```

---

## Validation at the boundary (controllers)

Bind the DTO with a param decorator; the framework validates before your handler runs, so the body is already typed and trusted inside the method.

```ts
@RestController('/projects')
export class ProjectController {
  @Post('/')
  async create(req: AuthenticatedRequest, _res: Response, @Body body: CreateProjectDto) {
    // `body` is validated + typed here — no manual parse, no `any`
    return await this.projectService.create(body);
  }

  @Get('/')
  async list(req, _res, @Query query: ListProjectsQueryDto) { … }
}
```

Manually, the same thing is:

```ts
const result = CreateProjectDto.safeParse(req.body);
if (!result.success) throw new BadRequestError(result.error.message);
const body = result.data;
```

**Rule:** validate **once, at the edge**. Past the controller, types are trusted — inner layers take typed DTOs/entities, never re-parse.

---

## Share one types package across FE and BE

All request/response DTOs and shared schemas live in a single package (`@n8n/api-types`) imported by *both* sides. The client builds requests against the same DTO the server validates with — the contract can't drift.

```ts
// frontend
import { CreateProjectDto } from '@n8n/api-types';
await api.post('/projects', new CreateProjectDto({ name }));   // typed payload

// backend (same import)
async create(@Body body: CreateProjectDto) { … }
```

---

## Where this fits vs. plain zod

| Use… | When |
|---|---|
| Bare `z.object` + `z.infer` ([zod.md](../cheatsheets/zod.md)) | Internal parsing, config, env vars, one-off validation |
| `schemas/` field schemas | A constraint reused across many endpoints (`projectNameSchema`) |
| `Z.class()` DTOs | A request/response contract bound to a route and shared FE↔BE |

> See also: [typescript-backend-architecture.md](typescript-backend-architecture.md), [typescript-error-handling.md](typescript-error-handling.md).
