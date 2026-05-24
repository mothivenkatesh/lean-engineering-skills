---
name: db-review
description: "Database schema, query, and access pattern review. Use when designing a schema, writing queries, reviewing slow endpoints, planning migrations, or evaluating whether to add a new database. Trigger on: 'design this schema', 'is this query good?', 'database is slow', 'should I add a cache', 'schema review', 'should I use MongoDB / Redis / Elasticsearch', 'help with migration', 'this query takes too long'. The database is the most expensive thing to get wrong — changing it after data exists costs more than everything else combined."
user-invocable: true
argument-hint: "[schema, query, or database question]"
---

# DB Review — Database Schema, Query, and Access Pattern Review

Kailash Nadh's principles from Zerodha (20TB, hundreds of billions of rows, 40,000–50,000 QPS on Postgres):

1. **Postgres handles 90% of use cases.** Add a specialized database only when Postgres measurably can't handle the workload.
2. **"My database is slow" is almost always bad indexing, bad queries, or bad schema.** Exhaust all Postgres-level optimizations before reaching for a new tool.
3. **Everyone forgets to index.** The most common performance failure is not a scaling problem — it's a missing index.
4. **Scale vertically before horizontally.** A single well-tuned Postgres instance handles billions of rows. Sharding is expensive, complex, and rarely necessary.

The discipline: **Fix the query before adding infrastructure. Fix the schema before adding a service.**

---

## THE DB REVIEW PROTOCOL

```
1. SCHEMA     → Is the data modeled correctly for this domain?
2. INDEXES    → Are reads going to be fast?
3. QUERIES    → Are queries correct and efficient?
4. ACCESS     → Are access patterns understood and optimized?
5. MIGRATION  → Is the migration safe?
6. TOOL       → Is this the right database for this data?
```

---

## DIMENSION 1: SCHEMA REVIEW

A bad schema is the most expensive engineering mistake. You can't fix schema without migrations, and migrations on large tables are expensive.

### Naming

- Table names: lowercase, snake_case, plural nouns (`orders`, `user_sessions`, not `Order`, `userSession`)
- Column names: lowercase, snake_case, descriptive (`created_at`, `payment_status`, not `ts`, `ps`)
- Boolean columns: `is_` or `has_` prefix (`is_verified`, `has_paid`, not `verified`, `paid`)
- Foreign keys: `[referenced_table]_id` (`user_id`, `order_id`)

If a column name doesn't tell you what it contains without reading the code, rename it.

### Data types

| Data | Correct type | Avoid |
|------|-------------|-------|
| Identifiers | UUID or serial BIGINT | VARCHAR for IDs |
| Money/currency | INTEGER (store as smallest unit: paise, cents) | FLOAT, DECIMAL for money |
| Timestamps | TIMESTAMPTZ (with timezone) | TIMESTAMP (no timezone) |
| Status/state | VARCHAR with CHECK constraint or enum | INTEGER status codes without table |
| JSON blobs | JSONB (binary, indexable) | JSON (text, slower) |
| IP addresses | INET type | VARCHAR |
| Flags/features | BOOLEAN | SMALLINT, VARCHAR('true') |

**Money as integers:** Never store money as FLOAT. `0.1 + 0.2 ≠ 0.3` in floating point. Store amounts as integers in the smallest unit (paise for INR, cents for USD). Convert for display only.

### Constraints

Every schema should enforce its invariants at the database level. Don't trust application code alone.

```sql
-- NOT NULL for columns that must always have a value
name VARCHAR(255) NOT NULL

-- CHECK for value constraints
status VARCHAR(50) CHECK (status IN ('pending', 'completed', 'failed'))

-- UNIQUE for uniqueness requirements
email VARCHAR(255) UNIQUE NOT NULL

-- FOREIGN KEY for referential integrity
user_id BIGINT REFERENCES users(id) ON DELETE CASCADE
```

Missing constraints = data integrity is "enforced" only by convention. Eventually, a bug or a script runs without the constraint, and you have corrupted data.

### Normalization (and when to denormalize)

**Normalize first.** Denormalize only when you have measured a performance problem that normalization causes.

Signs of under-normalization:
- Same data in multiple tables (update one, forget the other)
- Array columns storing entity IDs (join table should exist)
- JSON columns storing structured data that is queried (should be real columns)

Signs of over-normalization:
- Every join requires 5 tables (read queries are too complex)
- High-read, low-write data split into multiple tables unnecessarily

The rule: normalize for write integrity, denormalize for measured read performance.

---

## DIMENSION 2: INDEX REVIEW

**The most common database performance failure is a missing index.**

### When to add an index

Add an index on:
- **Every foreign key** (join columns). Postgres doesn't add these automatically.
- **Every column in a WHERE clause** on large tables
- **Every column in ORDER BY** on result sets you don't fully load
- **Composite index** when you always filter by A and B together: `CREATE INDEX ON orders (user_id, status)`

### Run EXPLAIN ANALYZE

Before and after any query optimization:

```sql
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 123 AND status = 'pending';
```

Read the output:
- **Seq Scan** on a large table = no index being used = slow
- **Index Scan** = index is being used = fast
- **Bitmap Index Scan** = index used for range query = acceptable
- **rows=10000 actual rows=3** = query planner estimate is wrong; analyze the table

```sql
ANALYZE orders;  -- update query planner statistics
```

### Index anti-patterns

| Anti-pattern | Problem |
|---|---|
| Index on boolean column | Low cardinality — Postgres often ignores it and seq scans anyway |
| Index on a column you always prepend with a function | `WHERE LOWER(email) = ?` doesn't use an index on `email` — create a functional index: `CREATE INDEX ON users (LOWER(email))` |
| Too many indexes | Each index slows writes; index only what's queried |
| Unused indexes | Check `pg_stat_user_indexes` — delete indexes with `idx_scan = 0` |
| No index on foreign keys | Every join does a seq scan on the child table |

### Partial indexes

For filtered queries, partial indexes are smaller and faster:

```sql
-- Only index active users (not all 10M users, just the 100K active ones)
CREATE INDEX ON users (email) WHERE is_active = true;

-- Only index pending orders
CREATE INDEX ON orders (created_at) WHERE status = 'pending';
```

---

## DIMENSION 3: QUERY REVIEW

### SELECT * is almost always wrong

```sql
-- WRONG: fetches all columns; network carries 10x the data you need
SELECT * FROM orders WHERE user_id = ?

-- RIGHT: fetch only what the application uses
SELECT id, status, total, created_at FROM orders WHERE user_id = ?
```

### Pagination without OFFSET

`OFFSET` on large tables is slow — Postgres reads and discards all rows before the offset.

```sql
-- WRONG: slow at large offsets
SELECT * FROM orders ORDER BY created_at DESC LIMIT 20 OFFSET 10000;

-- RIGHT: cursor-based pagination
SELECT * FROM orders WHERE created_at < $last_seen_cursor ORDER BY created_at DESC LIMIT 20;
```

### COUNT(*) on large tables

`COUNT(*)` does a seq scan or index scan over all rows. Alternatives:
- Use `pg_stat_user_tables.n_live_tup` for approximate counts (fast)
- Maintain a counter in a separate table, incremented on insert/delete
- Add `LIMIT 1` to existence checks: `SELECT 1 FROM orders WHERE user_id = ? LIMIT 1`

### N+1 in ORM code

```python
# WRONG (Django): 1 query for orders + N queries for each order's user
orders = Order.objects.all()
for order in orders:
    print(order.user.name)  # N queries

# RIGHT: prefetch in one query
orders = Order.objects.select_related('user').all()
```

Always check what queries your ORM generates. Log slow queries in development.

### Transactions and lock scope

Keep transactions short. The longer a transaction, the longer it holds locks, the more contention.

```sql
-- WRONG: transaction holds lock while doing non-DB work
BEGIN;
SELECT ... FOR UPDATE;  -- lock acquired
[application does 200ms of computation]
UPDATE ...;
COMMIT;  -- lock released after 200ms+

-- RIGHT: compute outside transaction, use narrow atomic update
[application does computation]
BEGIN;
UPDATE ... WHERE ...;  -- lock held for microseconds
COMMIT;
```

---

## DIMENSION 4: ACCESS PATTERN REVIEW

Schema follows access patterns. If the schema doesn't match how the application reads data, performance suffers.

**Questions to ask:**
1. What are the top 5 most frequent queries?
2. What are the top 3 slowest queries?
3. What queries are on the hot path (called on every request)?
4. What queries are read-heavy vs write-heavy?
5. What's the read/write ratio?

**Zerodha's pattern:** Hot path reads from Redis (in-memory), everything else from Postgres. Writes go to Postgres. Redis is refreshed asynchronously or on-demand. Result: 40ms mean latency at 40,000+ QPS.

**Cache invalidation strategy:**
- Write-through: write to DB and cache simultaneously (consistency, but write overhead)
- Cache-aside: read from cache; on miss, read from DB and populate cache (lazy, common)
- TTL-based: cache expires after N seconds (simple, but stale reads possible)

For the hot path: cache what changes rarely, TTL what changes occasionally, skip the cache for what changes on every write.

---

## DIMENSION 5: MIGRATION SAFETY

Schema migrations on tables with millions of rows are dangerous. PostgreSQL locks tables during many DDL operations.

### Safe vs unsafe migrations

| Operation | Safe? | Why |
|-----------|-------|-----|
| Adding a nullable column | Safe | No table rewrite |
| Adding a column with a default | **Unsafe on old Postgres** | Table rewrite in Postgres < 11; safe in 11+ |
| Adding NOT NULL without default | **Unsafe** | Must set value for all rows first |
| Adding an index | **Unsafe** (blocks writes) | Use `CREATE INDEX CONCURRENTLY` |
| Dropping a column | Safe (column still exists until vacuum) | Don't remove code references first |
| Renaming a column | **Unsafe** | Old column name disappears; app breaks |
| Changing column type | **Unsafe** | Table rewrite; blocks reads and writes |
| Adding a foreign key | **Unsafe** (table scan to validate) | Use `NOT VALID` then `VALIDATE CONSTRAINT` separately |

### The rename pattern (zero-downtime column rename)

```
Step 1: Add new column (e.g., add `payment_amount` alongside `amount`)
Step 2: Write to both columns in all code paths
Step 3: Backfill new column from old column
Step 4: Read from new column
Step 5: Remove writes to old column
Step 6: Drop old column
```

Never rename a column in a single migration in production. It breaks the running application.

### Large table migrations

For tables > 10M rows:
1. Use `CREATE INDEX CONCURRENTLY` — adds the index without blocking writes
2. Break data backfills into batches: update 1000 rows, sleep 10ms, repeat
3. Use `pg_repack` or `VACUUM FULL CONCURRENTLY` instead of `ALTER TABLE` for rewrites
4. Test migration duration on a production-sized copy first

---

## DIMENSION 6: TOOL SELECTION

**The Postgres Rule:** Use Postgres for everything until Postgres measurably can't handle it.

Postgres handles: relational data, JSON/JSONB, full-text search, geospatial (PostGIS), time series (TimescaleDB), vectors (pgvector), pub/sub (LISTEN/NOTIFY). Most "we need a specialized database" decisions are premature.

### When to reach for another tool

| Tool | Reach for it when |
|------|------------------|
| **Redis** | You need sub-millisecond reads; session storage; rate limiting; pub/sub; Postgres read latency is too high for hot path |
| **Elasticsearch** | Full-text search with typo tolerance, relevance ranking, facets — AND Postgres full-text is not meeting requirements |
| **ClickHouse** | Analytics queries over billions of rows with sub-second response |
| **Cassandra** | Write-heavy, time-series, globally distributed — AND Postgres can't handle the write volume |
| **S3 / Object storage** | Binary files, images, videos — never store blobs in Postgres |
| **Vector DB (Qdrant, Pinecone)** | Semantic search over millions of embeddings — AND pgvector is too slow |

**Before adding any new database:**
1. Have you added the right indexes to Postgres?
2. Have you run EXPLAIN ANALYZE on the slow queries?
3. Have you added a caching layer?
4. Have you tested Postgres with the right hardware (SSDs, more RAM)?
5. Can you solve this with a read replica?

Every new database is a new operational burden: new monitoring, new backup strategy, new failure mode, new expertise required. The team that runs 5 databases runs them all worse than the team that runs 1 well.

---

## REVIEW OUTPUT FORMAT

```
## Schema Review
[Tables and columns reviewed]

### Issues Found
| Issue | Severity | Location | Fix |
|-------|----------|----------|-----|
| Missing index on foreign key user_id | High | orders table | CREATE INDEX CONCURRENTLY |
| Money stored as FLOAT | Critical | payments.amount | Migrate to INTEGER (paise) |
| ...

## Query Review
[Queries analyzed]

### EXPLAIN ANALYZE Results
[Query → current plan → recommendation]

## Access Pattern Analysis
[Hot queries, read/write ratios, cache candidates]

## Migration Safety
[Migrations classified as safe/unsafe with safe alternatives]

## Tool Recommendation
[Is this the right database? If not, what and why]

## Priority Fixes
1. [Most critical — do immediately]
2. [High priority — next sprint]
3. [Medium — opportunistic]
```
