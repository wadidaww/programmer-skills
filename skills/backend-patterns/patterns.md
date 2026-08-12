# Skill: Backend Patterns

Reusable patterns for API design, database access, caching, messaging, and resilience.

---

## Repository Pattern

Abstracts data access behind an interface, making business logic independent of the database.

```typescript
// Interface — business logic depends on this, not on the DB
interface UserRepository {
  findById(id: string): Promise<User | null>
  findByEmail(email: string): Promise<User | null>
  save(user: User): Promise<User>
  delete(id: string): Promise<void>
}

// Implementation — swappable (Postgres, in-memory for tests, etc.)
class PostgresUserRepository implements UserRepository {
  constructor(private db: Database) {}

  async findById(id: string): Promise<User | null> {
    const row = await this.db.queryOne(
      'SELECT id, name, email FROM users WHERE id = $1',
      [id]
    )
    return row ? mapRowToUser(row) : null
  }
  // ...
}
```

---

## Service Layer Pattern

Orchestrates business logic, coordinates repositories and external services.

```typescript
class OrderService {
  constructor(
    private orders: OrderRepository,
    private inventory: InventoryService,
    private payments: PaymentGateway,
    private events: EventBus,
  ) {}

  async placeOrder(userId: string, items: OrderItem[]): Promise<Order> {
    await this.inventory.reserve(items)

    const order = Order.create({ userId, items })
    await this.payments.charge(order.total, userId)
    await this.orders.save(order)

    await this.events.publish(new OrderPlaced(order))
    return order
  }
}
```

---

## Unit of Work Pattern

Groups multiple operations into a single atomic transaction.

```typescript
interface UnitOfWork {
  users: UserRepository
  orders: OrderRepository
  commit(): Promise<void>
  rollback(): Promise<void>
}

class PlaceOrderHandler {
  async handle(cmd: PlaceOrderCommand): Promise<void> {
    const uow = await this.uowFactory.create()
    try {
      const user = await uow.users.findById(cmd.userId)
      const order = Order.create(user, cmd.items)
      await uow.orders.save(order)
      await uow.commit()
    } catch {
      await uow.rollback()
      throw
    }
  }
}
```

---

## CQRS Pattern

Separate command (write) and query (read) models.

```typescript
// Command side — optimised for writes
class CreateProductCommand {
  constructor(
    public readonly name: string,
    public readonly price: number,
    public readonly sku: string,
  ) {}
}

// Query side — optimised for reads (denormalised view)
interface ProductListView {
  id: string
  name: string
  price: number
  stockLevel: number
  categoryName: string  // denormalised — avoids a join at query time
}
```

---

## Outbox Pattern

Ensures database writes and event publishing are atomic — solves the dual-write problem.

```sql
-- Events are written to the database in the same transaction as the business data
CREATE TABLE outbox (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    topic       TEXT NOT NULL,
    payload     JSONB NOT NULL,
    created_at  TIMESTAMPTZ DEFAULT NOW(),
    published   BOOLEAN DEFAULT FALSE
);
```

```typescript
// In the same transaction: write business data + outbox event
await db.transaction(async (tx) => {
  await tx.query('INSERT INTO orders ...', [order])
  await tx.query(
    'INSERT INTO outbox (topic, payload) VALUES ($1, $2)',
    ['order.placed', JSON.stringify(orderPlacedEvent)]
  )
})
// A separate relay process reads the outbox and publishes to Kafka/SQS
```

---

## Circuit Breaker

Prevents cascading failures by stopping calls to an unhealthy dependency.

```typescript
import CircuitBreaker from 'opossum'

const breaker = new CircuitBreaker(callPaymentService, {
  timeout: 3000,           // fail if call takes > 3 seconds
  errorThresholdPercentage: 50, // open if 50% of calls fail
  resetTimeout: 10000,     // try again after 10 seconds
})

breaker.fallback(() => ({ status: 'deferred', message: 'Will retry shortly' }))
breaker.on('open', () => logger.warn('Payment service circuit open'))
```

---

## Retry with Exponential Back-off

```typescript
async function withRetry<T>(
  fn: () => Promise<T>,
  maxAttempts = 3,
  baseDelayMs = 100,
): Promise<T> {
  let lastError: Error
  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      return await fn()
    } catch (error) {
      lastError = error as Error
      if (attempt === maxAttempts) break
      const delay = baseDelayMs * 2 ** (attempt - 1) + Math.random() * 100 // jitter
      await sleep(delay)
    }
  }
  throw lastError!
}
```

---

## Pagination Patterns

### Cursor-Based (preferred for large, frequently updated datasets)

```typescript
// Request
GET /orders?cursor=eyJpZCI6IjEyMyJ9&limit=20

// Response
{
  "data": [...],
  "nextCursor": "eyJpZCI6IjE0MyJ9",
  "hasMore": true
}

// Query implementation
const orders = await db.query(
  `SELECT * FROM orders 
   WHERE id > $1 
   ORDER BY id 
   LIMIT $2`,
  [decodeCursor(cursor), limit + 1]
)
```

### Offset-Based (simple, suitable for small datasets or admin UIs)
```typescript
GET /orders?page=2&limit=20

// Works well but has "page drift" issues on frequently updated data
```

---

## Idempotency Keys

Make mutation endpoints safe to retry without side effects.

```typescript
// Client sends a unique idempotency key
POST /payments
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000

// Server: check if this key was already processed
const existing = await idempotencyStore.get(req.headers['idempotency-key'])
if (existing) return res.json(existing)  // return cached response

// Process and store result against the key
const result = await processPayment(req.body)
await idempotencyStore.set(req.headers['idempotency-key'], result, ttl=86400)
return res.json(result)
```

---

## Background Job Patterns

```typescript
// Bull/BullMQ example
const paymentQueue = new Queue('payment-processing', { connection: redis })

// Producer
await paymentQueue.add('charge', { orderId, amount }, {
  attempts: 3,
  backoff: { type: 'exponential', delay: 1000 },
  removeOnComplete: 1000,
  removeOnFail: 5000,
})

// Consumer
const worker = new Worker('payment-processing', async (job) => {
  await paymentService.charge(job.data.orderId, job.data.amount)
}, { connection: redis, concurrency: 5 })

worker.on('failed', (job, err) => {
  logger.error({ jobId: job?.id, err }, 'Payment job failed')
})
```
