---
name: db-review
description: "Database schema, query, and access pattern review. Run when designing a schema, writing queries, reviewing slow endpoints, planning migrations, or evaluating whether to add a new database. Trigger on: 'design this schema', 'is this query good?', 'database is slow', 'should I add a cache', 'schema review', 'should I use MongoDB/Redis/Elasticsearch', 'help with migration', 'this query takes too long'. The database is the most expensive thing to get wrong — changing it after data exists costs more than everything else combined."
user-invocable: true
argument-hint: "[schema, query, or database question]"
---

# DB Review — Schema, Query, and Access Pattern Review

## INPUT

Consumes: `SPEC.md` from `/spec` and `RISK.md` from `/premortem`

Uses: data model definitions, edge cases (for constraint design), failure modes related to data integrity.

Can also run standalone on an existing schema or query.

---

## THE LOOP

**TRIGGER:** Data model is defined in SPEC.md. Before writing any migration or schema code.

**CYCLE:**
1. Review schema naming, types, and constraints
2. Review indexes — find every missing one
3. Review queries — EXPLAIN ANALYZE every hot path query
4. Review access patterns — understand read/write ratios
5. Review migration safety — classify every change as safe or unsafe
6. Evaluate tool selection — is Postgres the right database?

**EXIT CONDITION:** Schema is correct, indexes are complete, queries are efficient, migrations are safe. SCHEMA.md is written.

---

## THE POSTGRES RULE

Use Postgres for everything until Postgres measurably can't handle it.

Zerodha: 20TB, hundreds of billions of rows, 40,000–50,000 QPS on Postgres. With 35 engineers.

"My database is slow" is almost always bad indexing, bad queries, or bad schema. Fix those before reaching for a new tool.

---

## DIMENSION 1: SCHEMA

A bad schema is the most expensive engineering mistake. You can't fix it without migrations. Migrations on large tables are expensive.

### Naming
- Tables: lowercase, snake_case, plural nouns (`orders`, `user_sessions`)
- Columns: lowercase, snake_case, descriptive (`created_at`, `payment_status`)
- Booleans: `is_` or `has_` prefix (`is_verified`, `has_paid`)
- Foreign keys: `[table]_id` pattern (`user_id`, `order_id`)

If the column name doesn't tell you what it contains without reading code: rename it.

### Data types

| Data | Type | Avoid |
|------|------|-------|
| Money | INTEGER (paise/cents) | FLOAT, DECIMAL |
| Timestamps | TIMESTAMPTZ | TIMESTAMP (no timezone) |
| IDs | UUID or BIGSERIAL | VARCHAR for IDs |
| Status/state | VARCHAR + CHECK constraint | Magic integers |
| JSON (queried) | JSONB | JSON (text, slower) |
| IP addresses | INET | VARCHAR |
| Booleans | BOOLEAN | SMALLINT, VARCHAR |

**Money as integers:** `0.1 + 0.2 ≠ 0.3` in floating point. Store INR as paise, USD as cents. Convert for display only. Never store money as FLOAT.

### Constraints

Enforce invariants at the database level. Don't trust application code alone.

```sql
name        VARCHAR(255) NOT NULL
status      VARCHAR(50)  CHECK (status IN ('pending', 'completed', 'failed'))
email       VARCHAR(255) UNIQUE NOT NULL
user_id     BIGINT       REFERENCES users(id) ON DELETE CASCADE
amount      INTEGER      CHECK (amount > 0)
```

Missing constraints = data integrity enforced only by convention. One bug, one script, and you have corrupted data.

### Normalization

Normalize first. Denormalize only when you have a measured performance problem normalization causes.

Under-normalized: same data in multiple tables; array columns storing IDs; JSON columns storing structured data that gets queried.

Over-normalized: every read requires 5 JOINs; high-read/low-write data split unnecessarily.

Rule: normalize for write integrity, denormalize for measured read performance.

---

## DIMENSION 2: INDEXES

**The most common database performance failure is a missing index.**

### Add an index on
- Every foreign key (Postgres doesn't add these automatically)
- Every column in a WHERE clause on large tables
- Every column in ORDER BY on unbounded result sets
- Composite: when you always filter by A and B together

```sql
CREATE INDEX ON orders (user_id);             -- foreign key
CREATE INDEX ON orders (user_id, status);     -- composite
CREATE INDEX CONCURRENTLY ON orders (email);  -- large tables, no downtime
```

### Run EXPLAIN ANALYZE on every query

```sql
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 123 AND status = 'pending';
```

Read the output:
- **Seq Scan** on a large table = no index = slow. Add an index.
- **Index Scan** = index used = fast.
- **Bitmap Index Scan** = range query = acceptable.
- `rows=10000 actual rows=3` = stale statistics. Run `ANALYZE orders`.

### Index anti-patterns

| Anti-pattern | Problem |
|---|---|
| Index on boolean column | Low cardinality — often ignored by planner |
| Index on `LOWER(col)` but no functional index | `WHERE LOWER(email) = ?` won't use index on `email` |
| Too many indexes | Each index slows writes |
| No index on foreign keys | Every join does a seq scan on the child table |

### Partial indexes (smaller, faster)

```sql
-- Only active users (not all 10M, just 100K active)
CREATE INDEX ON users (email) WHERE is_active = true;

-- Only pending orders
CREATE INDEX ON orders (created_at) WHERE status = 'pending';
```

---

## DIMENSION 3: QUERIES

### No SELECT *

```sql
-- WRONG: fetches all columns; carries 10x unnecessary data
SELECT * FROM orders WHERE user_id = ?

-- RIGHT: only what the application uses
SELECT id, status, total, created_at FROM orders WHERE user_id = ?
```

### Cursor-based pagination, not OFFSET

```sql
-- WRONG: slow at large offsets; reads and discards N rows
SELECT * FROM orders ORDER BY created_at DESC LIMIT 20 OFFSET 10000;

-- RIGHT: cursor-based
SELECT * FROM orders WHERE created_at < $cursor ORDER BY created_at DESC LIMIT 20;
```

### N+1 in ORM code

```python
# WRONG: 1 query + N queries for each user's orders
orders = Order.objects.all()
for order in orders:
    print(order.user.name)  # N queries

# RIGHT: one query with prefetch
orders = Order.objects.select_related('user').all()
```

Check what queries your ORM generates. Log slow queries in development.

### Short transactions

```sql
-- WRONG: lock held during non-DB work
BEGIN;
SELECT ... FOR UPDATE;   -- lock acquired
[application does 200ms computation]
UPDATE ...;
COMMIT;                  -- lock released after 200ms+

-- RIGHT: compute outside transaction
[application does computation]
BEGIN;
UPDATE ... WHERE ...;    -- lock held microseconds
COMMIT;
```

---

## DIMENSION 4: ACCESS PATTERNS

Schema follows access patterns. If schema doesn't match how the application reads, performance suffers.

Answer before finalizing schema:
1. Top 5 most frequent queries?
2. Top 3 slowest queries?
3. Which queries are on the hot path (every request)?
4. Read-heavy or write-heavy?
5. Read/write ratio?

**Zerodha's pattern:** Hot path reads from Redis. Everything else from Postgres. Writes go to Postgres. Redis refreshed asynchronously. Result: 40ms mean latency at 40,000+ QPS.

Cache invalidation strategy:
- **Cache-aside:** Read from cache; on miss, read from DB and populate. Common.
- **Write-through:** Write to DB and cache simultaneously. Consistent, more write overhead.
- **TTL-based:** Cache expires after N seconds. Simple; stale reads possible.

---

## DIMENSION 5: MIGRATION SAFETY

PostgreSQL locks tables during many DDL operations. Migrations on tables with millions of rows are dangerous.

| Operation | Safe? | Note |
|-----------|-------|------|
| Add nullable column | Safe | No table rewrite |
| Add column with default (Postgres 11+) | Safe | |
| Add NOT NULL without default | **Unsafe** | Must backfill first |
| Add index | **Use CONCURRENTLY** | Blocks writes without it |
| Drop column | Safe (data not immediately removed) | Remove code refs first |
| Rename column | **Unsafe** | Old name disappears; app breaks |
| Change column type | **Unsafe** | Table rewrite; blocks reads and writes |
| Add foreign key | **Unsafe** | Table scan to validate; use NOT VALID first |

### Zero-downtime column rename (3 deploys, never 1)

```
Deploy 1: Add new column. Write to both old and new. Read from old.
Deploy 2: Read from new. Stop writing to old.
Deploy 3: Drop old column.
```

Never rename in one migration in production.

### Large table migrations (> 10M rows)

- `CREATE INDEX CONCURRENTLY` — adds index without blocking writes
- Backfills in batches: update 1000 rows, sleep 10ms, repeat
- Test migration duration on a production-sized copy first

---

## DIMENSION 6: TOOL SELECTION

**Use Postgres for everything until Postgres measurably can't handle it.**

Postgres handles: relational data, JSONB, full-text search, geospatial (PostGIS), time series (TimescaleDB), vectors (pgvector), pub/sub (LISTEN/NOTIFY).

| Add this | Only when |
|----------|-----------|
| Redis | Sub-millisecond reads; session storage; rate limiting; Postgres hot-path latency is measured too high |
| Elasticsearch | Full-text with typo tolerance and relevance ranking — AND Postgres full-text is insufficient |
| ClickHouse | Analytics over billions of rows, sub-second response |
| Object storage (S3) | Binary files, images, video — never blobs in Postgres |

Before adding any new database, answer all five:
1. Added the right indexes to Postgres?
2. Run EXPLAIN ANALYZE on slow queries?
3. Added a caching layer?
4. Tested Postgres with proper hardware?
5. Tried a read replica?

Every new database is a new operational burden: new monitoring, new backup strategy, new failure mode.

---

## OUTPUT: SCHEMA.md

```markdown
# SCHEMA.md

## Schema Review

### Tables
[Entity name, fields, types, constraints, indexes]

### Issues Found
| Issue | Severity | Location | Fix |
|-------|----------|----------|-----|
| Missing index on foreign key user_id | High | orders | CREATE INDEX CONCURRENTLY |
| Money stored as FLOAT | Critical | payments.amount | Migrate to INTEGER (paise) |

## Query Review

### EXPLAIN ANALYZE Results
[Query → current plan → recommendation]

## Access Pattern Analysis
[Hot queries, read/write ratios, cache candidates]

## Migration Safety
[Each migration: safe or unsafe, with safe alternative]

## Tool Recommendation
[Is Postgres sufficient? If not, what and why]

## Priority Fixes
1. [Critical — do immediately]
2. [High — next sprint]
3. [Medium — opportunistic]

## Feeds into
/modular (MODULE_MAP.md) — module boundaries inform which service owns which tables
```

---

## FEEDS INTO

`/modular` — which modules own which tables, and what their data access interfaces look like.
