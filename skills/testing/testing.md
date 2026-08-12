# Skill: Testing

Comprehensive guide to software testing at every level.

---

## Testing Pyramid

```
             ┌────────────┐
             │    E2E     │  ← 5–10 tests (critical journeys)
             └────────────┘
           ┌────────────────┐
           │  Integration   │  ← 50–200 tests (service boundaries)
           └────────────────┘
         ┌────────────────────┐
         │     Unit Tests     │  ← 500+ tests (all logic)
         └────────────────────┘
```

- **Unit tests** are fast (< 10 ms each), isolated, and numerous.
- **Integration tests** test the interaction between components (DB, HTTP, message queue).
- **E2E tests** test the entire system from the user's perspective.

---

## Test Design Principles

### Arrange-Act-Assert
```typescript
describe('OrderService.placeOrder', () => {
  it('emits OrderPlaced event after successful order', async () => {
    // Arrange
    const eventBus = createMockEventBus()
    const service = new OrderService(mockOrderRepo, mockInventory, mockPayments, eventBus)
    const items = [{ productId: 'p1', quantity: 2 }]

    // Act
    await service.placeOrder('user-1', items)

    // Assert
    expect(eventBus.publish).toHaveBeenCalledWith(
      expect.objectContaining({ type: 'OrderPlaced', userId: 'user-1' })
    )
  })
})
```

### Test Naming
Good test names form a readable sentence:
- `should return 404 when user does not exist`
- `given insufficient stock, throws InsufficientStockError`
- `when payment fails, order is not saved`

### One Assertion Per Concept
Each test should verify one logical behaviour. Multiple `expect` calls are fine if they test the same concept.

---

## Unit Testing Patterns

### Mocking
```typescript
// Use jest.fn() / vi.fn() to mock dependencies
const mockUserRepo: jest.Mocked<UserRepository> = {
  findById: jest.fn(),
  save: jest.fn(),
}

mockUserRepo.findById.mockResolvedValue(null)

// Verify interactions
expect(mockUserRepo.findById).toHaveBeenCalledWith('user-1')
expect(mockUserRepo.findById).toHaveBeenCalledTimes(1)
```

### Test Doubles
| Type | Description | When to Use |
|---|---|---|
| **Stub** | Returns predefined responses | When you only need a return value |
| **Mock** | Records calls + verifiable expectations | When you need to verify an interaction |
| **Fake** | Simplified real implementation | In-memory repository for integration tests |
| **Spy** | Wraps real object, records calls | When you want to observe a real object |

### Parameterised Tests
```typescript
// Vitest / Jest
describe.each([
  ['', false],
  ['   ', false],
  ['alice@example.com', true],
  ['not-an-email', false],
])('isValidEmail(%s)', (input, expected) => {
  it(`returns ${expected}`, () => {
    expect(isValidEmail(input)).toBe(expected)
  })
})
```

---

## Integration Testing

### TestContainers (Docker-based)
```typescript
import { PostgreSqlContainer } from '@testcontainers/postgresql'

let container: StartedPostgreSqlContainer

beforeAll(async () => {
  container = await new PostgreSqlContainer('postgres:16').start()
  await runMigrations(container.getConnectionUri())
}, 30_000)

afterAll(async () => {
  await container.stop()
})

it('saves and retrieves a user', async () => {
  const repo = new PostgresUserRepository(container.getConnectionUri())
  const user = await repo.save({ name: 'Alice', email: 'alice@example.com' })
  const found = await repo.findById(user.id)
  expect(found?.email).toBe('alice@example.com')
})
```

### API Integration Tests
```typescript
import supertest from 'supertest'

const request = supertest(app)

it('returns 401 for unauthenticated requests', async () => {
  const res = await request.get('/api/orders')
  expect(res.status).toBe(401)
})

it('returns the user orders for authenticated requests', async () => {
  const token = await getTestToken('user-1')
  const res = await request
    .get('/api/orders')
    .set('Authorization', `******
  
  expect(res.status).toBe(200)
  expect(res.body.data).toBeInstanceOf(Array)
})
```

---

## E2E Testing with Playwright

```typescript
import { test, expect } from '@playwright/test'

test.describe('Checkout flow', () => {
  test.beforeEach(async ({ page }) => {
    await seedTestData()
    await page.goto('/login')
    await page.getByLabel('Email').fill('test@example.com')
    await page.getByLabel('Password').fill('password')
    await page.getByRole('button', { name: 'Sign in' }).click()
  })

  test('user can complete a purchase', async ({ page }) => {
    await page.goto('/products')
    await page.getByTestId('product-card').first().click()
    await page.getByRole('button', { name: 'Add to cart' }).click()
    await page.goto('/cart')
    await page.getByRole('button', { name: 'Checkout' }).click()

    await expect(page.getByText('Order confirmed')).toBeVisible()
    await expect(page).toHaveURL(/\/orders\/[a-z0-9-]+/)
  })
})
```

---

## Test Data Management

### Factories / Builders
```typescript
// test/factories/user.ts
import { faker } from '@faker-js/faker'

export function buildUser(overrides: Partial<User> = {}): User {
  return {
    id: faker.string.uuid(),
    name: faker.person.fullName(),
    email: faker.internet.email(),
    createdAt: new Date(),
    ...overrides,
  }
}

// Usage
const user = buildUser({ email: 'alice@example.com' })
const admin = buildUser({ role: 'admin' })
```

### Database Seeding
```typescript
// Seed once before the test suite; truncate before each test
beforeEach(async () => {
  await db.query('TRUNCATE orders, order_items CASCADE')
})
```

---

## Snapshot Testing

Use sparingly — only for stable, small outputs:
```typescript
it('renders the order card correctly', () => {
  const { container } = render(<OrderCard order={buildOrder()} />)
  expect(container.firstChild).toMatchSnapshot()
})
```

Avoid snapshot tests for:
- Large or complex output.
- Frequently changing UI.
- Business logic (use explicit assertions instead).

---

## Performance Testing

```javascript
// k6 load test
import http from 'k6/http'
import { check, sleep } from 'k6'

export const options = {
  stages: [
    { duration: '1m', target: 100 },   // ramp up
    { duration: '5m', target: 100 },   // sustain
    { duration: '1m', target: 0 },     // ramp down
  ],
  thresholds: {
    http_req_duration: ['p(99)<500'],   // 99th percentile < 500ms
    http_req_failed: ['rate<0.01'],     // < 1% error rate
  },
}

export default function () {
  const res = http.get('https://api.example.com/orders')
  check(res, { 'status is 200': (r) => r.status === 200 })
  sleep(1)
}
```

---

## Coverage Guidelines

- Target **≥ 80 %** line and branch coverage for production code.
- Focus coverage on business logic; do not write trivial tests to inflate numbers.
- Use coverage to find gaps, not as a KPI to optimise.
- Exclude: generated code, config files, migrations, `index.ts` re-exports.

```json
// jest.config.json coverage thresholds
{
  "coverageThreshold": {
    "global": {
      "branches": 80,
      "functions": 80,
      "lines": 80
    }
  }
}
```
