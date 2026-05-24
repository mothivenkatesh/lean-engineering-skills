---
name: hot-path
description: "Hot path review and optimization for latency-sensitive code. Use when reviewing code that runs on every request, every user interaction, or in tight loops. Trigger on: 'this is slow', 'optimize this endpoint', 'review for performance', 'this runs on every request', 'latency is too high', 'p99 is bad', 'database calls in a loop'. The hot path is sacred — everything that touches it pays a tax in every request, forever."
user-invocable: true
argument-hint: "[code or endpoint to review]"
---

# Hot Path — Latency-Sensitive Code Review

## INPUT

Consumes: `TEST_REPORT.md` from `/tdd`

Uses: confirmation that behaviors are correct before optimizing. Don't optimize untested code.

Can also run standalone on any existing endpoint.

---

## THE LOOP

**TRIGGER:** Tests are green (TEST_REPORT.md). Code works. Now make it fast.

**CYCLE:**
1. Identify what's actually on the hot path (not everything is)
2. Measure baseline (p50, p95, p99) — never optimize without a baseline
3. Check each dimension in order (network → memory → disk → DB → computation → parallelism)
4. Apply fixes
5. Measure again — quantify the improvement

**EXIT CONDITION:** No N+1 queries, no unbounded queries, no sequential calls that could be parallel, no disk I/O on hot path, all external calls have timeouts. PERF.md is written.

**The discipline:** Zerodha. 40ms mean latency. 40,000+ QPS. Postgres + Redis + Go. 35 engineers. No Kubernetes. The hot path is sacred.

---

## HARD CONSTRAINTS
- **Required:** Working, tested code. TEST_REPORT.md should confirm tests pass. Do not optimize untested code — correctness before performance.
- **Refuse if:** Tests are not passing. Fix the tests first.
- **Mandatory:** Measure baseline (p50, p95, p99) before any optimization. Never recommend a fix without a measured baseline to compare against.
- **Never assume the bottleneck.** Identify it with profiling or EXPLAIN ANALYZE. "It's probably the database" is not a diagnosis.
- **Refuse if:** Any external HTTP call has no explicit timeout. Flag it as a blocker before reviewing anything else.

---

## WHAT IS THE HOT PATH?

Hot path characteristics:
- Runs synchronously during a user-facing request
- Called thousands to millions of times per day
- Users wait for it to complete
- Its latency determines the product's perceived speed

Not the hot path:
- Batch jobs running at night
- Analytics and reporting queries
- Admin dashboards
- Background workers
- Webhook handlers with async delivery

Only the hot path needs hot-path treatment. Applying it everywhere is over-engineering. Ignoring it on the actual hot path is negligence.

---

## DIMENSION 1: NETWORK CALLS

Every network call on the hot path is a latency multiplier you don't control.

### N+1 queries
```python
# WRONG: 1 query + N queries for each user's orders
users = db.query("SELECT * FROM users")
for user in users:
    orders = db.query("SELECT * FROM orders WHERE user_id = ?", user.id)

# RIGHT: one query
orders = db.query("SELECT * FROM orders WHERE user_id IN (?)", [u.id for u in users])
```

### Queries inside loops
Any `db.query()`, `redis.get()`, `http.call()` inside a `for` loop is N calls. Batch fetch before the loop.

### Sequential calls that could be parallel

```python
# WRONG: total time = A + B + C
a = fetch_user(id)
b = fetch_permissions(id)
c = fetch_preferences(id)

# RIGHT: total time = max(A, B, C)
a, b, c = await asyncio.gather(fetch_user(id), fetch_permissions(id), fetch_preferences(id))
```

### External HTTP calls without timeouts

```python
response = requests.get(url, timeout=0.5)  # 500ms hard limit — always set this
```

No external call on the hot path without a timeout. Library defaults are often 30 seconds or infinite. A 30-second hung request blocks a thread. At 100 concurrent users, this exhausts your thread pool.

---

## DIMENSION 2: MEMORY AND ALLOCATION

Every allocation is eventually garbage collected. At high throughput, GC pauses become latency spikes.

### String concatenation in loops
```go
// WRONG: allocates a new string on every iteration
for _, item := range items {
    result += fmt.Sprintf("%s,", item.Name)
}

// RIGHT: pre-allocate, build once
builder := strings.Builder{}
builder.Grow(estimatedSize)
for _, item := range items {
    builder.WriteString(item.Name)
}
```

### Fetching more data than needed
```sql
-- WRONG: fetches all columns; network carries 10x the data needed
SELECT * FROM orders WHERE user_id = ?

-- RIGHT: only what the application uses
SELECT id, status, total FROM orders WHERE user_id = ?
```

Return the minimum data the caller needs. Nothing more.

---

## DIMENSION 3: DISK I/O

Disk is orders of magnitude slower than memory. Nothing on the hot path should touch disk if it can serve from memory.

**Kailash Nadh's principle:** On hot paths, everything serves from memory. Disk is for durability, not reads.

Look for:
- File reads on every request (config files, static data — load once at startup)
- Reads that could be cached in Redis or in-process memory
- Synchronous logging that blocks (use async logging on hot paths)
- Session data on disk instead of Redis

### Caching hierarchy
```
In-process memory  → ~nanoseconds  (fastest; limited size; per-instance)
Redis / Memcache   → ~1ms          (fast; shared; survives restarts)
Database           → ~5-50ms       (authoritative; durable)
Disk               → ~10-100ms     (avoid on hot path)
External API       → ~50-500ms     (last resort; add timeout + fallback)
```

Move reads as high in this hierarchy as data size, change frequency, and consistency requirements allow.

---

## DIMENSION 4: DATABASE QUERIES

Before adding caching or scaling the database: fix the query.

For every DB query on the hot path:

1. **Run EXPLAIN ANALYZE.** Seq Scan on a large table = add an index.
2. **Is the index selective?** Index on a boolean is useless — half the rows match.
3. **Is there a LIMIT?** Any query without LIMIT can return 1 row or 1 million.
4. **Are transactions necessary?** Transactions hold locks. Only use for atomic operations.
5. **Is the connection pool sized correctly?** Too small = requests queue. Too large = DB overwhelmed. Typical: 10-20 connections per app instance.

```sql
EXPLAIN ANALYZE SELECT id, status FROM orders WHERE user_id = 123 AND status = 'pending';
-- Seq Scan on orders → add index
-- Index Scan → good
```

---

## DIMENSION 5: COMPUTATION

Expensive computation on the hot path runs in every request.

Look for:
- Regex compiled inside a request handler (compile once at startup, reuse)
- Repeated calculation of the same value (compute once, cache the result)
- Sorting large in-memory collections (use the right data structure)
- CPU-intensive work blocking the event loop (move to background worker)
- Deserializing the same data repeatedly (cache the deserialized form)

---

## DIMENSION 6: PARALLELISM

Latency = the sum of sequential steps. Parallelism converts sequential latency into the maximum of parallel steps.

Identify the critical path: which operations depend on each other? Which are independent?

```javascript
// WRONG: sequential despite async
const user = await getUser(id);
const perms = await getPermissions(id);
const prefs = await getPreferences(id);
// Total: user + perms + prefs

// RIGHT: parallel
const [user, perms, prefs] = await Promise.all([
  getUser(id),
  getPermissions(id),
  getPreferences(id)
]);
// Total: max(user, perms, prefs)
```

---

## MEASURE BEFORE AND AFTER

Never claim a performance fix works without measurement.

Before touching code:
1. Measure current latency (p50, p95, p99) under realistic load
2. Identify the bottleneck with profiling — don't guess
3. Document the baseline

After:
1. Measure again under same conditions
2. Quantify: not "faster" — "p99 dropped from 450ms to 80ms"
3. Confirm no other metric degraded (memory, CPU, error rate)

Profiling tools:
- Python: `cProfile`, `py-spy`
- Node.js: `--prof`, clinic.js
- Go: `pprof`
- General: Jaeger (distributed tracing), Grafana

---

## HOT PATH CHECKLIST

```
Network:
[ ] No N+1 queries
[ ] No queries inside loops
[ ] Independent calls parallelized
[ ] All external HTTP calls have explicit timeouts
[ ] Circuit breakers on flaky dependencies

Memory:
[ ] No unnecessary allocations in loops
[ ] SELECT only needed columns
[ ] Results are paginated (LIMIT applied)

Caching:
[ ] Hot reads from Redis or in-process cache
[ ] Cache invalidation is correct
[ ] Cache misses don't cause thundering herd

Database:
[ ] EXPLAIN ANALYZE run on every query — no unexpected Seq Scans
[ ] Indexes exist for all filter/sort/join columns
[ ] Connection pool sized correctly

Computation:
[ ] Regex compiled at startup, not per-request
[ ] CPU-heavy work moved off hot path
[ ] No repeated deserialization of same data

Concurrency:
[ ] Independent operations run in parallel
[ ] No unnecessary blocking
```

---

## OUTPUT: PERF.md

```markdown
# PERF.md

## Hot Path Identified
[What code/endpoint is on the hot path]

## Baseline
p50: Xms | p95: Xms | p99: Xms  (measured before any changes)

## Bottleneck Analysis
[Rank issues by impact: what costs the most latency?]

## Issues Found
| Issue | Type | Estimated Impact | Location |
|-------|------|-----------------|----------|
| N+1 query on users | Network | High (50ms × N) | user_service.py:42 |

## Fixes Applied
### Fix 1: [Issue name]
Before: [code or metric]
After: [code or metric]

## Result
p50: Xms | p95: Xms | p99: Xms  (measured after changes)
Improvement: [p99 dropped from Xms to Yms]

## Feeds into
/ops-ready (DEPLOY.md) — performance baseline becomes part of the ops readiness check
```

---

## FEEDS INTO

`/ops-ready` — PERF.md provides the latency baseline for the four golden signals. Observability is built around these numbers.
