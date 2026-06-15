# TypeScript Backend Architecture

**What it is:** A convention set for structuring a large TypeScript/Node backend — dependency injection, Controller→Service→Repository layering, decorator-based routing, and a strict file-naming system.

**Why people use it:** Keeps a big codebase navigable and testable — every file's role is obvious from its name, dependencies are explicit and mockable, and HTTP/business/data concerns stay separated.

**Typically used for:** Express/Nest-style API servers, monorepo backends, anything where many engineers touch the same code and units must be unit-testable in isolation.

> Patterns below are grounded in [n8n](https://github.com/n8n-io/n8n) (`packages/cli`, `packages/@n8n/di`, `packages/@n8n/decorators`), a large production TS monorepo.

---

## File naming — role suffixes

Every file declares its role with a dot-suffix, kebab-case, **one class per file**, file named after the class.

```
role.controller.ts      RoleController        HTTP routes, no logic
role.service.ts         RoleService           business logic
role.repository.ts      RoleRepository        DB access
create-role.dto.ts      CreateRoleDto         request shape (validated)
project.schema.ts       projectNameSchema…    reusable zod field schemas
not-found.error.ts      NotFoundError         one error type
role.service.test.ts    —                     co-located test
__tests__/              —                     or a sibling test dir
```

Grep-friendly: `*.service.ts` finds every service; opening `role.controller.ts` tells you exactly what it is before you read a line.

---

## Dependency injection

A class is made injectable with a decorator; dependencies are declared as `private readonly` constructor params and resolved by the container. No manual wiring, no singletons-by-import.

```ts
import { Service } from '@n8n/di';

@Service()
export class RoleService {
  constructor(
    private readonly roleRepository: RoleRepository,
    private readonly roleCacheService: RoleCacheService,
    private readonly logger: Logger,
  ) {}

  async findAll() {
    return await this.roleRepository.find();
  }
}
```

Resolve at an entry point (the container builds the whole dependency graph):

```ts
import { Container } from '@n8n/di';
const roleService = Container.get(RoleService);
```

**Why it matters for tests:** constructor injection means a unit test just passes mocks — no module mocking, no globals.

```ts
const roleRepository = mock<RoleRepository>();
const service = new RoleService(roleRepository, mock(), mock());
```

> n8n ships its own tiny IoC container ([`@n8n/di`](https://github.com/n8n-io/n8n/blob/master/packages/%40n8n/di/src/di.ts)) — `@Service()` registers the class, `Container.get()` lazily instantiates and caches, and it detects circular dependencies. The same shape works with `tsyringe` or NestJS DI.

---

## The Controller → Service → Repository layers

Each layer has one job. Logic never leaks up into the controller or down into the repository.

| Layer | Responsibility | Must NOT |
|---|---|---|
| **Controller** | Parse/validate request, call a service, return result | Contain business logic or touch the DB |
| **Service** | Business rules, orchestration, transactions | Know about HTTP (req/res, status codes) |
| **Repository** | DB queries only | Contain business rules |

```ts
// role.controller.ts — thin: validate in, delegate, return
@RestController('/roles')
export class RoleController {
  constructor(private readonly roleService: RoleService) {}

  @Post('/')
  async create(req: AuthenticatedRequest, _res: Response, @Body body: CreateRoleDto) {
    return await this.roleService.create(body);   // no logic here
  }
}
```

The service throws **domain** errors; the controller (or a global handler) maps them to HTTP. The service stays HTTP-agnostic, so it's reusable from a CLI, a queue worker, etc.

---

## Decorator-based routing

Routes are declared with decorators next to the handler, not in a separate route table — the method signature *is* the contract. Param decorators bind and validate.

```ts
import { RestController, Get, Post, Patch, Body, Param, Query } from '@n8n/decorators';

@RestController('/api-keys')
export class ApiKeysController {
  constructor(private readonly apiKeysService: ApiKeysService) {}

  @Post('/')
  async create(req: AuthenticatedRequest, _res: Response, @Body body: CreateApiKeyRequestDto) { … }

  @Get('/')
  async list(req: AuthenticatedRequest, _res: Response, @Query query: ListApiKeysQueryDto) { … }

  @Patch('/:id')
  async update(req, _res, @Param('id') id: string, @Body body: UpdateApiKeyRequestDto) { … }
}
```

`@Body`/`@Query`/`@Param` bind the DTO and run validation at the boundary (see the DTO/validation sheet). Authorization is layered with more decorators:

```ts
@Get('/', { middlewares: [rateLimit] })
@Licensed('feat:advancedPermissions')
@Scoped('role:read')
async list(...) {}
```

---

## TypeScript rules that hold the structure together

These are the rules that keep the layers honest. (n8n enforces them via ESLint.)

```ts
// ❌ never `any` — use `unknown` and narrow
function handle(input: any) {}
function handle(input: unknown) { if (typeof input === 'string') … }

// ❌ avoid `as` casts — use a type guard / predicate instead
const u = data as User;
function isUser(x: unknown): x is User { … }     // then: if (isUser(data)) …
// (`as` is acceptable only in test code)

// shared FE↔BE types live in ONE package, imported by both sides
import type { RoleAssignmentsResponse } from '@n8n/api-types';
```

**Lazy-load heavy modules.** If a module is only used on one code path (a big parser, a native addon), import it at point of use, not at the top:

```ts
async function exportPdf() {
  const { renderPdf } = await import('./heavy-pdf-renderer');  // not loaded until needed
  return renderPdf();
}
```

---

## Request lifecycle

```
HTTP request
  → Controller         @Body validates DTO, extracts req.user
    → Service          business rules; throws UserError / domain errors
      → Repository     DB query
    ← entity/data
  ← Controller         returns plain object (serialized to JSON)
  ↑ domain error → mapped to ResponseError (404/400/…) by error layer
```

Each arrow is a layer you can unit-test in isolation by mocking the layer below.

> See also: [typescript-error-handling.md](typescript-error-handling.md), [zod-dtos-and-validation-boundaries.md](zod-dtos-and-validation-boundaries.md), [typescript-testing-patterns.md](typescript-testing-patterns.md).
