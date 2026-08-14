# Skill: Performance

Profiling, optimisation techniques, and performance engineering principles.

---

## Performance Engineering Process

1. **Define targets**: set SLOs (latency p99, throughput, error rate) before optimising.
2. **Measure**: profile with real production workloads or representative benchmarks.
3. **Identify bottlenecks**: use flame graphs to find the hot path.
4. **Optimise**: change one thing at a time.
5. **Validate**: verify the improvement with the same benchmark.
6. **Monitor**: track regressions in CI and production.

---

## Profiling Tools

| Platform | CPU | Memory | Latency |
|---|---|---|---|
| Node.js | `--prof`, `clinic.js` | `heapdump`, `v8.writeHeapSnapshot` | `clinic doctor` |
| Python | `cProfile`, `py-spy`, `scalene` | `memory-profiler`, `tracemalloc` | `pyinstrument` |
| Go | `pprof`, `go tool trace` | `pprof` heap | `go bench` |
| JVM | async-profiler, JFR | JProfiler, MAT | JMH |
| Rust | `cargo flamegraph`, `samply` | `heaptrack` | `criterion` |
| Browser | Chrome DevTools Performance, Lighthouse | Memory tab | WebPageTest |

---

## Database Performance

### Query Optimisation
```sql
-- Use EXPLAIN ANALYSE to understand query plans
EXPLAIN (ANALYSE, BUFFERS, FORMAT TEXT)
SELECT u.id, u.name, COUNT(o.id) AS order_count
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
WHERE u.created_at > NOW() - INTERVAL '30 days'
GROUP BY u.id;
```

Key things to look for in query plans:
- **Seq Scan on large table** → needs an index.
- **Nested loop with many rows** → consider a hash join.
- **High buffer hits vs reads** → data is cached; low reads is good.
- **Sort** → add an index that matches the ORDER BY.

### Indexing
```sql
-- Composite index: most selective column first
CREATE INDEX idx_orders_user_status ON orders (user_id, status)
WHERE status != 'delivered';  -- partial index excludes old data

-- Covering index: include all columns needed by the query
CREATE INDEX idx_orders_user_covering ON orders (user_id)
INCLUDE (id, total, created_at);
```

### Connection Pooling
- Use PgBouncer (PostgreSQL) or ProxySQL (MySQL) between the app and database.
- Set pool size = (CPU cores × 2) + effective_spindle_count (rule of thumb).
- Monitor pool wait time; scale up if > 5 ms average.

---

## Caching

### Cache-Aside (Lazy Loading)
```typescript
async function getUser(id: string): Promise<User> {
  const cached = await redis.get(`user:${id}`)
  if (cached) return JSON.parse(cached)

  const user = await db.users.findById(id)
  if (!user) throw new NotFoundError(`User ${id} not found`)
  
  await redis.set(`user:${id}`, JSON.stringify(user), 'EX', 300)
  return user
}
```

### When to Cache
- Cache when: reads >> writes, data is expensive to compute, data is shared across requests.
- Do NOT cache: user-specific sensitive data without encryption, highly personalised data, data that must be strongly consistent.

### Cache Sizing
- Cache 20 % of the working set to capture 80 % of requests (Pareto principle).
- Use `redis-cli --latency` and `INFO stats` to monitor hit rate and eviction rate.
- Target cache hit rate > 90 % for read-heavy workloads.

---

## Async Processing

Move expensive work off the request path:
```typescript
// BAD — blocking the request for 5 seconds
app.post('/users', async (req, res) => {
  const user = await createUser(req.body)
  await sendWelcomeEmail(user)      // blocks for 500ms
  await generateAvatarThumbnail(user) // blocks for 4.5s
  res.json(user)
})

// GOOD — return immediately, process async
app.post('/users', async (req, res) => {
  const user = await createUser(req.body)
  await emailQueue.add('welcome', { userId: user.id })
  await imageQueue.add('avatar', { userId: user.id })
  res.status(201).json(user)  // responds in < 50ms
})
```

---

## Node.js Performance

```typescript
// Profile with --prof
node --prof server.js
// Process profile
node --prof-process isolate-*.log > profile.txt

// Use worker threads for CPU-intensive work
import { Worker } from 'worker_threads'

app.post('/compress', async (req, res) => {
  const result = await runInWorker('./workers/compress.js', req.body)
  res.json(result)
})

// Avoid synchronous file I/O on the hot path
// BAD
const data = fs.readFileSync('config.json')
// GOOD — read once at startup
const config = JSON.parse(await fs.readFile('config.json', 'utf8'))
```

---

## Frontend Performance

### JavaScript Bundle Optimisation
```typescript
// Dynamic import for route-level code splitting
const HeavyComponent = React.lazy(() => import('./HeavyComponent'))

// Analyse bundle size
// npx webpack-bundle-analyzer stats.json
// npx vite-bundle-visualizer
```

### Image Optimisation
- Use WebP/AVIF format (30–50 % smaller than JPEG/PNG).
- Use `srcset` for responsive images.
- Use `loading="lazy"` for below-the-fold images.
- Use a CDN with image resizing (Cloudflare, Imgix, Vercel Image Optimization).

```html
<img
  src="hero.webp"
  srcset="hero-480.webp 480w, hero-960.webp 960w, hero-1920.webp 1920w"
  sizes="(max-width: 480px) 480px, (max-width: 960px) 960px, 1920px"
  alt="Hero image"
  loading="eager"
  fetchpriority="high"
/>
```

### Rendering Performance
- Avoid layout thrashing: batch DOM reads before writes.
- Use `will-change: transform` for elements that animate frequently.
- Debounce scroll and resize event handlers.
- Use `content-visibility: auto` for off-screen sections.
- Virtualise long lists (react-window, @tanstack/virtual).

---

## Benchmarking Checklist

- [ ] Benchmark runs on production-representative data and traffic patterns.
- [ ] JIT/cache warm-up done before measuring.
- [ ] Results reported as histograms (p50, p95, p99, p99.9), not averages.
- [ ] CPU frequency scaling disabled; benchmark machine is isolated.
- [ ] Baseline measured before the change; comparison uses the same conditions.
- [ ] Improvement validated with statistical significance (≥ 5 runs, p < 0.05).
- [ ] Regression test added to CI to prevent future regressions.
