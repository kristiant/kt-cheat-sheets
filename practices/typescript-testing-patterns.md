# TypeScript Testing Patterns

**What it is:** Opinionated conventions for writing unit tests in a typed backend — typed mocks, shared fixtures, isolated units, and HTTP mocking.

**Why people use it:** Tests stay fast, type-safe, and readable; mocking deps is trivial because the code is built around constructor injection.

**Typically used for:** Unit-testing services, controllers, and utilities in isolation. For framework API differences (Jest vs Vitest), see [jest-vitest.md](../cheatsheets/jest-vitest.md).

> Grounded in [n8n](https://github.com/n8n-io/n8n) — Jest + [`jest-mock-extended`](https://github.com/marchaos/jest-mock-extended), `nock` for HTTP.

---

## Typed mocks with `mock<T>()`

`jest-mock-extended`'s `mock<T>()` returns a fully-typed mock where every method is a `jest.fn()` — autocomplete works, and the compiler catches it if the real interface changes.

```ts
import { mock } from 'jest-mock-extended';

const roleRepository = mock<RoleRepository>();
roleRepository.findById.mockResolvedValue(role);   // typed: arg + return checked
```

Prefer this over hand-rolled `as unknown as RoleRepository` casts — those silently rot when the interface changes.

---

## Isolating units via constructor injection

Because services take their deps as constructor args, a unit test just passes mocks — no module mocking, no globals. Mock everything below the unit under test; `mock()` (no type arg) covers deps you don't assert on.

```ts
describe('ActiveWorkflowsService', () => {
  const workflowRepository = mock<WorkflowRepository>();
  const sharedWorkflowRepository = mock<SharedWorkflowRepository>();
  const service = new ActiveWorkflowsService(
    mock(),                      // logger — unused here
    workflowRepository,
    sharedWorkflowRepository,
    mock(),
    mock(),
  );

  beforeEach(() => jest.clearAllMocks());   // reset call history between tests

  it('filters out workflows with activation errors', async () => {
    // Arrange
    workflowRepository.getActiveIds.mockResolvedValue(['1', '2', '3']);
    // Act
    const ids = await service.getAllActiveIdsInStorage();
    // Assert
    expect(ids).toEqual(['2', '3']);
  });
});
```

---

## Shared, hoisted fixtures

Define immutable typed mocks **once** at the describe scope and reuse them. This matters for speed: `jest-mock-extended` builds nested proxy objects, and recreating a complex `mock<User>()` in every test adds up fast.

```ts
describe('RoleService', () => {
  const user = mock<User>();          // built once, reused
  const repo = mock<RoleRepository>();
  const service = new RoleService(repo, mock(), mock());

  beforeEach(() => jest.clearAllMocks());  // clear calls, keep the mock objects
  // tests mutate behavior per-case via repo.xxx.mockResolvedValue(...)
});
```

`jest.clearAllMocks()` resets call history without throwing away the (expensive) mock instances. Don't replace these shared typed mocks with `as unknown as T` shortcuts — you lose the type contract.

---

## Test structure (AAA, describe per method)

Nest a `describe` per method under test, an `it` per behavior. Read tests as a spec of the unit.

```ts
describe('RoleService', () => {
  describe('create', () => {
    it('throws BadRequestError when the slug already exists', async () => {
      repo.findBySlug.mockResolvedValue(existingRole);
      await expect(service.create(dto)).rejects.toThrow(BadRequestError);
    });

    it('persists and returns the new role', async () => { … });
  });
});
```

Assert on **error type**, not message text (`rejects.toThrow(BadRequestError)`) — messages change, types are the contract. Verify collaborators when the interaction is the behavior:

```ts
expect(sharedWorkflowRepository.getSharedWorkflowIds).toHaveBeenCalledWith(activeIds);
expect(sharedWorkflowRepository.getSharedWorkflowIds).not.toHaveBeenCalled();
```

---

## Mock the network with `nock`

For code that makes HTTP calls, intercept at the network layer rather than mocking the client — you exercise the real request-building code.

```ts
import nock from 'nock';

afterEach(() => nock.cleanAll());

it('fetches the user', async () => {
  nock('https://api.example.com').get('/users/1').reply(200, { id: 1, name: 'Ada' });

  const user = await client.getUser(1);
  expect(user.name).toBe('Ada');
});

// fail the test if any real HTTP escapes the mocks:
nock.disableNetConnect();
```

---

## Mocking the LLM in agent code

Treat the model like any external dependency: inject it, mock its response, and test *your* logic (routing, tool selection, output parsing) deterministically — no API calls, no flakiness, no cost.

```ts
const model = mock<ChatModel>();
model.invoke.mockResolvedValue({ content: '{"action":"search","query":"weather"}' });

const decision = await agent.decideNextStep(state);
expect(decision.tool).toBe('search');   // asserts your parsing, not the model
```

---

## Rules of thumb

- **Mock all external dependencies** in a unit test (DB, network, other services, clock).
- **One assertion focus per `it`** — name it after the behavior, not the method.
- **`as` is acceptable in test code** (and only there) — but prefer `mock<T>()` for whole objects.
- **Reset between tests** (`jest.clearAllMocks()` in `beforeEach`) so call history doesn't leak.
- **Typecheck the tests too** — typed mocks only help if `tsc` runs over the test files.

> See also: [jest-vitest.md](../cheatsheets/jest-vitest.md) (framework API), [typescript-backend-architecture.md](typescript-backend-architecture.md) (why DI makes this easy).
