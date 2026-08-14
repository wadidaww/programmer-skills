# Skill: System Design

Architecture patterns, principles, and decision frameworks for designing scalable and reliable systems.

---

## The Design Process

1. **Clarify requirements** — functional (what the system does) and non-functional (scale, latency, availability).
2. **Define the API** — the external contract before the internals.
3. **Design the data model** — entities, relationships, access patterns.
4. **High-level design** — components, data flows, and boundaries.
5. **Deep-dive** — the hard parts: bottlenecks, failure modes, scaling strategies.
6. **Trade-off analysis** — why this design over alternatives.

---

## Capacity Estimation

| Metric | Formula |
|---|---|
| QPS | Users × requests/user/day ÷ 86400 |
| Storage/year | rows/day × row_size × 365 |
| Bandwidth | QPS × average_response_size |
| Memory (cache) | QPS × cache_hit_rate × object_size |

**Useful numbers to memorise:**
- L1 cache: 1 ns. L2: 4 ns. RAM: 100 ns. SSD: 100 µs. HDD: 10 ms. Network (same DC): 0.5 ms. Cross-continent: 150 ms.
- 1 Gbps NIC can handle ~100 MB/s sustained.
- A single PostgreSQL instance handles ~10 000 simple QPS.
- Redis handles ~100 000 QPS on a single node.

---

## Scalability Patterns

### Horizontal Scaling
- Add more instances behind a load balancer.
- Requires stateless services; externalise session state to Redis or a database.
- Use consistent hashing to distribute load without full redistribution on scale events.

### Database Scaling
- **Read replicas**: route read traffic to replicas; primary handles only writes.
- **Vertical scaling**: larger instance — limited and expensive.
- **Sharding**: partition data across nodes by a shard key.
  - Choose a shard key with high cardinality and uniform distribution.
  - Avoid shard keys that create hot spots (e.g. timestamp-based keys for write-heavy workloads).
- **Connection pooling**: use PgBouncer or ProxySQL to reduce connection overhead.

### Caching Strategy
```
Request → CDN cache (static/semi-static)
       → Application cache (Redis) — read-through or cache-aside
       → Database
```

Cache invalidation strategies:
- **TTL-based**: simple, risk of stale data.
- **Event-based invalidation**: publish a cache invalidation event on every mutation.
- **Write-through**: update cache and DB synchronously on write.
- **Write-behind (write-back)**: update cache immediately; async flush to DB.

---

## Common System Design Patterns

### URL Shortener
- Generate a short code (base62 of a counter or random 7 chars).
- Store `short_code → original_url` in a key-value store (Redis + DynamoDB).
- Return `301 Moved Permanently` (cacheable) or `302 Found` (analytics-preserving).
- Rate limit creation; check destination URL safety.

### News Feed / Timeline
- **Fan-out on write**: on post, push to all followers' feed tables. Fast reads, slow writes, expensive for celebrities.
- **Fan-out on read**: compute feed at read time. Slow reads, fast writes.
- **Hybrid**: fan-out on write for normal users; fan-out on read for high-follower accounts.

### Distributed Rate Limiter
- Use the **token bucket** or **sliding window** algorithm.
- Store state in Redis with Lua scripts for atomic read-increment-expire.
- Apply at: per-user, per-IP, per-endpoint levels.

```lua
-- Redis Lua script for token bucket (atomic)
local key = KEYS[1]
local capacity = tonumber(ARGV[1])
local refill_rate = tonumber(ARGV[2])
local now = tonumber(ARGV[3])

local data = redis.call('HMGET', key, 'tokens', 'last_refill')
-- ... (full algorithm omitted for brevity)
```

### Notification System
- **Producers**: user action services publish events to a queue (Kafka, SQS).
- **Notification service**: consumes events, formats notifications, routes to channels.
- **Channels**: push (FCM/APNs), email (SendGrid), SMS (Twilio), in-app (WebSocket).
- **Delivery tracking**: store notification state; retry on failure; track opens/clicks.

---

## Database Selection Guide

| Use Case | Recommended | Reasoning |
|---|---|---|
| Transactional (OLTP) | PostgreSQL | ACID, rich SQL, JSON, extensions |
| Document store | MongoDB, DynamoDB | Flexible schema, horizontal scale |
| Wide-column (IoT, time-series) | Cassandra, Bigtable | High write throughput, linear scale |
| Time-series | TimescaleDB, InfluxDB | Optimised for temporal queries |
| Search | Elasticsearch, OpenSearch | Full-text, faceted search |
| Graph | Neo4j, Amazon Neptune | Relationship traversals |
| Cache / session | Redis | Sub-ms reads, data structures |
| Analytics (OLAP) | BigQuery, Snowflake, ClickHouse | Columnar, MPP, large scans |
| Message queue | Kafka | Durable, high-throughput event streaming |

---

## Consistency Patterns

### Strong Consistency
- All reads see the latest write.
- Requires coordination (quorums, 2PC, Raft/Paxos).
- Higher latency; use for financial data, inventory.

### Eventual Consistency
- Reads may see stale data; system converges over time.
- Higher availability and lower latency.
- Use for social feeds, recommendations, non-critical data.

### Read-Your-Writes Consistency
- A user always sees their own writes immediately.
- Route reads for a user's own data to the primary; route other reads to replicas.

---

## High Availability Design

- Deploy across ≥ 3 availability zones.
- Use health checks with graceful removal from load balancer rotation.
- Circuit break failing dependencies.
- Design for zero-downtime deployments (rolling/blue-green/canary).
- Define and test RTO and RPO targets.

---

## Microservices Communication

### Synchronous (REST/gRPC)
- Use for user-facing requests where a response is needed immediately.
- Set timeouts on every call; implement circuit breakers.
- Use gRPC for internal service-to-service calls (binary, type-safe, streaming).

### Asynchronous (Events)
- Use for decoupled workflows where the caller does not need an immediate response.
- Use for fan-out (one event → many consumers).
- Use Kafka for durable, high-throughput event streaming.
- Use SQS/RabbitMQ for reliable task queues.

### Saga Pattern (Distributed Transactions)
- Coordinate multi-service transactions without 2PC.
- **Choreography**: each service reacts to events and emits its own events.
- **Orchestration**: a saga orchestrator sends commands to services and handles failures.
- Use compensating transactions to roll back partial failures.

---

## Design Tradeoffs Reference

| Dimension | Tradeoff |
|---|---|
| Consistency vs Availability | CAP theorem — partition forces a choice |
| Latency vs Throughput | Batching increases throughput but adds latency |
| Storage vs Compute | Precompute and cache expensive results |
| Simplicity vs Scalability | Microservices add operational complexity |
| Flexibility vs Performance | Dynamic schemas (document DBs) vs fixed schemas (relational) |
| Cost vs Reliability | Multi-region is more reliable but 2–3× more expensive |
