---
name: hot-path
description: "Hot path review and optimization for latency-sensitive code. Use when reviewing or writing code that runs on every request, every user interaction, or in tight loops. Trigger on: 'this is slow', 'optimize this endpoint', 'review this for performance', 'this runs on every request', 'latency is too high', 'p99 is bad', 'database calls in a loop', 'high throughput'. The hot path is sacred — everything that touches it pays a tax in every request, forever."
user-invocable: true
argument-hint: "[code or endpoint to review]"
---

# Hot Path — Latency-Sensitive Code Review

Kailash Nadh's principle: **The hot path is sacred.**

Zerodha's SLA: 40ms mean latency across the entire system, serving 15-20% of India's daily stock trades. With 35 engineers. No Kubernetes. The discipline that makes this possible is treating the hot path as inviolable.

The hot path is any code that executes on every user request, every transaction, or in a tight loop. Every millisecond wasted on the hot path is wasted for every user, every time. The cost compounds silently until it doesn't.

---

## WHAT IS THE HOT PATH?

Before reviewing anything, identify what's actually hot.

**Hot path characteristics:**
- Runs synchronously during a user-facing request
- Called thousands to millions of times per day
- Users wait for it to complete
- Its latency directly determines the product's perceived speed

**Not the hot path:**
- Batch jobs running at night
- Analytics and reporting queries
- Admin dashboards
- Background workers processing queues
- Webhook handlers with async delivery

**Rule:** Only the hot path needs hot-path treatment. Applying hot-path optimization everywhere is over-engineering. Ignoring it on the actual hot path is negligence.

---

## THE HOT PATH REVIEW

Read the code, then check each dimension in order.

---

## DIMENSION 1: NETWORK CALLS

Every network call on the hot path is a latency multiplier you don't control.

**Check for:**

### N+1 queries
The most common hot path killer. Pattern:
```python
# WRONG: 1 query to get users + N queries for each user's orders
users = db.query("SELECT * FROM users")
for user in users:
    orders = db.query("SELECT * FROM orders WHERE user_id = ?", user.id)
```

Fix: JOIN or batch load with `IN (...)`. One round trip, not N+1.

### Queries inside loops
Any `db.query()`, `redis.get()`, `http.call()` inside a `for` loop is N queries. Replace with:
- Batch fetch before the loop
- Single query with JOIN
- `IN (...)` clause with collected IDs

### Sequential calls that could be parallel
```python
# WRONG: Sequential — total time = A + B + C
a = fetch_user(id)
b = fetch_permissions(id)
c = fetch_preferences(id)

# RIGHT: Parallel — total time = max(A, B, C)
a, b, c = await asyncio.gather(fetch_user(id), fetch_permissions(id), fetch_preferences(id))
```

### Synchronous HTTP calls to external services
Every external HTTP call on the hot path is a dependency you don't control. Timeouts are your only protection. Always set them:
```python
response = requests.get(url, timeout=0.5)  # 500ms hard limit
```

Flag any external HTTP call without a timeout. Flag any external HTTP call without a circuit breaker.

---

## DIMENSION 2: MEMORY AND ALLOCATION

On hot paths, every allocation is work. Every object created is eventually garbage-collected. At high throughput, GC pauses become latency spikes.

**Check for:**

### Byte/string conversions in loops
```go
// WRONG: allocates a new string on every iteration
for _, item := range items {
    result += fmt.Sprintf("%s,", item.Name)  // repeated allocation
}

// RIGHT: pre-allocate and build once
builder := strings.Builder{}
builder.Grow(estimatedSize)
for _, item := range items {
    builder.WriteString(item.Name)
    builder.WriteByte(',')
}
```

### Creating large objects inside loops
Move object creation outside the loop when possible. Reuse buffers.

### Loading more data than needed
```sql
-- WRONG: fetches all columns; application uses 3
SELECT * FROM orders WHERE user_id = ?

-- RIGHT: fetch only what you need
SELECT id, status, total FROM orders WHERE user_id = ?
```

Return the minimum data the caller needs. Nothing more.

---

## DIMENSION 3: DISK I/O

Disk is orders of magnitude slower than memory. Nothing on the hot path should touch disk if it can serve from memory.

**Zerodha's principle:** On hot paths, everything serves from memory. Disk is for durability, not reads.

**Check for:**
- File reads on every request (config files, templates, static data)
- Database reads that could be cached in Redis or in-process memory
- Logging that blocks (use async logging on hot paths)
- Session data stored on disk instead of Redis

**Caching hierarchy for hot paths:**
```
In-process memory  → ~nanoseconds  (fastest, limited size, per-instance)
Redis / Memcache   → ~1ms          (fast, shared, survives restarts)
Database           → ~5-50ms       (slower, authoritative, durable)
Disk               → ~10-100ms     (avoid on hot path)
External API       → ~50-500ms     (last resort; add timeout + fallback)
```

Move as many reads as possible up this hierarchy. The right cache layer is the one that fits the data's size, change frequency, and consistency requirements.

---

## DIMENSION 4: DATABASE QUERIES

"My database is slow" is almost always bad indexing, bad queries, or bad schema. Before adding caching or scaling the database, fix the query.

**Mandatory check for any DB query on the hot path:**

1. **Does it use an index?** Run `EXPLAIN ANALYZE`. If you see "Seq Scan" on a large table, add an index.

2. **Is the index selective?** An index on a boolean column is useless — half the rows match. Indexes on high-cardinality columns (IDs, emails, timestamps) are valuable.

3. **Are you fetching unbounded results?** Any query without a `LIMIT` can return 1 row or 1 million. Add limits.

4. **Are you using transactions where you don't need them?** Transactions hold locks. On the hot path, only use transactions for operations that must be atomic.

5. **Connection pool configured correctly?** Too small = requests queue. Too large = DB overwhelmed. Typical: 10-20 connections per app instance.

**The index rule:** Index every column you filter on, every column you sort on, and every foreign key you join on. Then measure. Indexes make reads fast; they make writes slightly slower.

---

## DIMENSION 5: COMPUTATION

Expensive computation on the hot path is avoidable work that runs in every request.

**Check for:**
- Regex compilation inside a request handler (compile once at startup, reuse)
- Repeated calculation of the same value (compute once, cache the result)
- Sorting or searching large in-memory collections (use the right data structure)
- Blocking the event loop with CPU-intensive work (move to background worker)
- Deserializing the same data repeatedly (cache the deserialized form)

---

## DIMENSION 6: PARALLELISM AND CONCURRENCY

Latency = the sum of sequential steps. Parallelism converts sequential latency into the maximum of parallel steps.

**Identify the critical path:** Draw the sequence of operations. Which ones depend on each other? Which ones are independent?

Independent operations can run in parallel:
- Fetch user data + fetch permissions + fetch feature flags → all independent → parallelize
- Validate input → then query DB → then write → sequential (each depends on previous)

**Async/await mistakes on hot paths:**
```javascript
// WRONG: sequential despite async
const user = await getUser(id);
const perms = await getPermissions(id);
const prefs = await getPreferences(id);
// total time: user + perms + prefs

// RIGHT: parallel
const [user, perms, prefs] = await Promise.all([
  getUser(id),
  getPermissions(id),
  getPreferences(id)
]);
// total time: max(user, perms, prefs)
```

---

## MEASURING BEFORE AND AFTER

Never claim a performance fix works without measurement.

**Before touching code:**
1. Measure current latency (p50, p95, p99) under realistic load
2. Identify the bottleneck with profiling (don't guess)
3. Document the baseline

**After the fix:**
1. Measure again under the same conditions
2. Quantify the improvement (not "faster" — "p99 dropped from 450ms to 80ms")
3. Check that no other metric degraded (memory, CPU, error rate)

**Profiling tools:**
- Python: `cProfile`, `py-spy`
- Node.js: `--prof`, clinic.js
- Go: `pprof`
- Ruby: `ruby-prof`, `stackprof`
- General: distributed tracing (Jaeger, Tempo), APM (Datadog, Grafana)

---

## HOT PATH REVIEW CHECKLIST

```
Network:
[ ] No N+1 queries
[ ] No queries inside loops
[ ] Independent calls parallelized
[ ] All external HTTP calls have timeouts
[ ] Circuit breakers on flaky dependencies

Memory:
[ ] No unnecessary allocations in loops
[ ] SELECT only needed columns
[ ] Results are paginated (LIMIT applied)

Caching:
[ ] Hot reads served from Redis or in-process cache
[ ] Cache invalidation is correct (not over-aggressive, not stale)
[ ] Cache misses don't cause thundering herd

Database:
[ ] EXPLAIN ANALYZE run on every query — no unexpected Seq Scans
[ ] Indexes exist for all filter/sort/join columns
[ ] Connection pool sized appropriately

Computation:
[ ] Regex compiled at startup, not per-request
[ ] CPU-heavy work moved off the hot path
[ ] No repeated deserialization of the same data

Concurrency:
[ ] Independent operations run in parallel
[ ] No unnecessary blocking

Logging:
[ ] No synchronous log writes on hot path
[ ] Log level is appropriate (not DEBUG in production)
```

---

## OUTPUT FORMAT

```
## Hot Path Identified
[What code/endpoint/function is on the hot path]

## Bottleneck Analysis
[Rank issues by impact: what's costing the most latency?]

## Issues Found
| Issue | Type | Estimated Impact | Location |
|-------|------|-----------------|----------|
| N+1 query on users | Network | High (50ms × N) | user_service.py:42 |
| ...

## Fixes (in priority order)
### Fix 1: [Issue name]
[Code before → code after + explanation]

### Fix 2: [Issue name]
[Code before → code after + explanation]

## Expected Result
[Estimated latency improvement if all fixes are applied]

## Measurement Plan
[How to verify the improvement: what to measure, before and after]
```
