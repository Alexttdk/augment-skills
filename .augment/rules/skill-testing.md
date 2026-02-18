---
type: agent_requested
description: Apply when writing unit/integration/E2E tests, designing test strategy, analyzing coverage gaps, reviewing test quality, or setting up CI/CD quality gates. General methodology only — not browser-specific.
---

# Testing Methodology

## When to Use
- Writing unit, integration, E2E, or property-based tests
- Designing test strategy for a feature or system
- Analyzing coverage gaps, flaky tests, or test quality issues
- Setting up automation frameworks or CI/CD quality gates
- Code reviews that include test evaluation

> **Scope boundary**: This skill = general testing methodology. For browser/Playwright/Cypress specifics → use `skill-web-testing.md`.

## Testing Pyramid & Strategy Selection

```
          ╔═══════╗
          ║  E2E  ║  10-20% — critical user journeys, slow
         ╔╩═══════╩╗
         ║  Integ  ║  20-30% — service contracts, DB, API
        ╔╩═════════╩╗
        ║   Unit    ║  50-70% — logic, algorithms, fast, isolated
        ╚═══════════╝
```

| Test Type | Use When | Don't Use When |
|-----------|----------|----------------|
| Unit | Isolated logic, pure functions, algorithms | Testing integrations or side effects |
| Integration | API contracts, DB queries, cross-service | Mocking every dependency (defeats the purpose) |
| E2E | Critical user journeys, full system flows | Every code path (too slow, brittle) |
| Property-based | Input validation, parsers, encoders | Simple deterministic functions |
| Mutation | Checking test suite completeness | CI pipeline (run offline on schedule) |

## TDD Iron Laws

> **NO PRODUCTION CODE WITHOUT A FAILING TEST FIRST.**

### Red-Green-Refactor Cycle

```typescript
// 1. RED: Write minimal failing test — OBSERVE the failure
it('rejects empty email', () => {
  expect(validateEmail('')).toBe(false);
}); // RUN → ✗ FAIL: validateEmail is not defined

// 2. GREEN: Simplest code to pass — nothing extra
function validateEmail(email: string): boolean {
  return email.length > 0 && email.includes('@');
} // RUN → ✓ PASS

// 3. REFACTOR: Improve while tests stay green. Then repeat.
```

**Iron rules:**
- A test you've never seen fail proves nothing
- Bug fix workflow: write test exposing the bug FIRST → then fix
- "I'll add tests later" = tests never get added

## Unit Testing Patterns

### AAA: Arrange → Act → Assert

```typescript
describe('UserService', () => {
  let service: UserService;
  let mockRepo: jest.Mocked<UserRepository>;

  beforeEach(() => {
    mockRepo = { findById: jest.fn(), save: jest.fn() } as any;
    service = new UserService(mockRepo);
  });
  afterEach(() => jest.clearAllMocks());

  it('returns user when found', async () => {
    mockRepo.findById.mockResolvedValue({ id: '1', name: 'Alice' }); // Arrange
    const result = await service.getUser('1');                        // Act
    expect(result).toEqual({ id: '1', name: 'Alice' });              // Assert
    expect(mockRepo.findById).toHaveBeenCalledWith('1');
  });

  it('throws NotFoundError when user missing', async () => {
    mockRepo.findById.mockResolvedValue(null);
    await expect(service.getUser('1')).rejects.toThrow(NotFoundError);
  });
});
```

### Test Organization
```typescript
describe('Feature', () => {
  describe('happy path', () => { /* expected behavior */ });
  describe('edge cases', () => { /* empty, max, boundary inputs */ });
  describe('error cases', () => { /* invalid input, system failures */ });
});
```

## Integration Testing

Use real DB/services in test environment. Mock only external third-party services.

```typescript
// API integration (Supertest)
describe('POST /api/users', () => {
  it('creates user with valid data', async () => {
    const res = await request(app)
      .post('/api/users').send({ email: 'test@test.com', name: 'Test' })
      .expect(201);
    expect(res.body.email).toBe('test@test.com');
    expect(res.body.id).toBeDefined();
  });

  it('returns 400 for invalid email', async () => {
    await request(app)
      .post('/api/users').send({ email: 'invalid' })
      .expect(400);
  });
});

// DB integration
describe('UserRepository', () => {
  beforeEach(async () => await db.query('DELETE FROM users'));
  afterAll(async () => await db.end());

  it('creates and retrieves user', async () => {
    const user = await userRepo.create({ email: 'x@x.com', name: 'X' });
    expect(await userRepo.findById(user.id)).toEqual(user);
  });
});
```

## E2E Strategy

Target **critical user journeys only** — keep count low (20-50 max).

```
✓ Test with E2E:  registration, login/logout, core business transaction
✗ Not E2E:        every validation rule (unit), admin edge cases (integration)
```

**Principles:**
- Use stable selectors: roles, labels, test-ids — not CSS classes
- Page Object Model or Screenplay pattern for maintainability
- Run on PR merge, not every commit
- Test behavior, not implementation details

## Mocking Patterns — When & How

| What to Mock | When | Don't Mock |
|-------------|------|-----------|
| External APIs / payment gateways | Always (unstable, costs money) | Internal business logic |
| Database (in unit tests) | Yes — mock repo layer | In integration tests (use real test DB) |
| Time / clock | When testing time-dependent logic | Randomly |
| File system | I/O-heavy unit tests | When testing actual file operations |
| Message queues | Unit level | Integration tests (use real queue) |

**Mock completeness rule:** Mock responses must mirror the real API structure. Incomplete mocks hide production bugs.

```typescript
// ✅ Complete mock using factory
const createMockUser = (overrides = {}) => ({
  id: '1', name: 'Alice', email: 'alice@example.com',
  createdAt: '2024-01-01T00:00:00Z', permissions: ['read', 'write'],
  ...overrides,
});
mockRepo.findById.mockResolvedValue(createMockUser({ id: '42' }));

// ❌ Incomplete mock — crashes in production when code accesses .email
mockRepo.findById.mockResolvedValue({ id: '42', name: 'Alice' });
```

**Spy vs mock vs stub:**
```typescript
jest.fn()                          // Mock — replace & control
jest.spyOn(obj, 'method')          // Spy — wrap real impl, assert calls
jest.mock('./module', () => (...)) // Stub — replace entire module
```

## Coverage Guidance

| Metric | Excellent | Good | Needs Work |
|--------|-----------|------|------------|
| Line coverage | > 90% | 70–90% | < 70% |
| Branch coverage | > 85% | 65–85% | < 65% |
| Defect leakage | < 2% | 2–5% | > 5% |
| Automation ratio | > 80% | 60–80% | < 60% |

**Coverage targets by layer:**
- Unit: 80–90% (fast, cheap to write)
- Integration: 60–70% (key contracts and error paths)
- E2E: 20–30% (critical paths only)

**Coverage ≠ quality**: 100% line coverage with trivial assertions is worthless. Assert specific values, not just that code ran.

### Feedback Cycle Targets
```
Unit tests:        < 5 min   (run on file save / pre-commit)
Integration tests: < 15 min  (run on commit / PR open)
E2E tests:         < 30 min  (run on PR merge)
Full regression:   < 2 hours (nightly)
```

## Anti-Patterns Catalog

### 1. Testing Mock Behavior (not real behavior)
```typescript
// ❌ Only verifies mock was called — no behavior tested
expect(mockApi).toHaveBeenCalledWith(1);

// ✅ Assert on actual component output
expect(await service.getUser(1)).toMatchObject({ name: 'Alice' });
```

### 2. Test-Only Methods in Production Code
```typescript
// ❌ _reset() only exists for tests — pollutes production class
class Cache { _resetForTesting() { this.data.clear(); } }

// ✅ Use fresh instances per test (beforeEach)
beforeEach(() => { cache = new Cache(); });
```

### 3. Over-Mocking (Mock Without Understanding)
```typescript
// ❌ Everything mocked — what did we actually test?
jest.mock('./inventory'); jest.mock('./payment'); jest.mock('./shipping');

// ✅ Use real internals; mock only external boundaries
const inventory = new InventoryService(testDb); // Real — catches real bugs
const payment = mockPaymentGateway();            // External third-party = mocked
```

### 4. Incomplete Mocks
```typescript
// ❌ Minimal stub passes unit test but crashes production
mockUser.mockResolvedValue({ id: 1 }); // Missing .email, .permissions → runtime error

// ✅ Factory ensures mock matches full production shape
mockUser.mockResolvedValue(createMockUser({ id: 1 }));
```

### 5. Tests as Afterthought
```
// "Ship now, add tests later" = tests never get written
// No feature is done until its tests ship with it (TDD enforces this)
```

## Quality Gates (CI/CD)

```markdown
## Release Gate — all must pass:
- [ ] Zero critical defects
- [ ] Line coverage ≥ 80%
- [ ] All unit + integration tests green
- [ ] No new flaky tests introduced
- [ ] Security scan clean
- [ ] Performance regression < 10% vs baseline
```

**Shift-Left benefits:** Defects caught in unit tests cost ~10× less to fix than defects found in production.

## Constraints

**MUST DO:**
- Test happy paths AND all error cases
- Mock external dependencies; use real components in integration tests
- Write descriptive test names: `it('throws NotFoundError when user is missing')`
- Assert specific values — not just that mocks were called
- Write tests before marking features complete (TDD)
- Run full suite in CI/CD; fix flaky tests immediately

**MUST NOT:**
- Skip error-case testing
- Use production data in tests
- Write order-dependent tests (each test must be independently runnable)
- Test implementation details (test behavior, not internals)
- Leave debug code or `.only` / `.skip` in committed test files
- Mock everything in integration tests

## References
- Source: https://github.com/Jeffallan/claude-skills/tree/main/skills/test-master

