# Index Design Reference

## Table of Contents
- [Index Selection Philosophy](#index-selection-philosophy)
- [B-Tree Indexes](#b-tree-indexes)
- [Skip Scan (PG18+)](#skip-scan-pg18)
- [Composite Indexes](#composite-indexes)
- [Covering Indexes](#covering-indexes)
- [Partial Indexes](#partial-indexes)
- [Expression Indexes](#expression-indexes)
- [GIN Indexes](#gin-indexes)
- [GiST Indexes](#gist-indexes)
- [BRIN Indexes](#brin-indexes)
- [pgvector Indexes](#pgvector-indexes)
- [Index Maintenance](#index-maintenance)
- [Anti-Patterns](#anti-patterns)

---

## Index Selection Philosophy

Index for **access patterns you actually query**, not for coverage. Every index:
- Speeds up reads that match its key structure
- Slows down every `INSERT`, `UPDATE`, and `DELETE` on the table
- Consumes storage and requires vacuum maintenance

Design indexes backwards from your top queries. Use `pg_stat_user_indexes` to identify and drop unused indexes.

### Quick Selection Guide

| Access Pattern | Index Type | Example |
|---|---|---|
| Equality, range, ORDER BY | **B-tree** | `WHERE id = ?`, `WHERE created_at > ?` |
| JSONB containment, key existence | **GIN** | `WHERE data @> '{"status":"active"}'` |
| Array containment/overlap | **GIN** | `WHERE tags @> ARRAY['pg']` |
| Full-text search | **GIN** | `WHERE search_vector @@ query` |
| Range overlap, exclusion | **GiST** | `WHERE period && '[2026-01-01,2026-02-01)'` |
| Spatial/geometry | **GiST** | PostGIS queries |
| Time-series, append-only | **BRIN** | `WHERE created_at BETWEEN ...` on ordered data |
| Vector similarity | **HNSW / IVFFlat** | `ORDER BY embedding <=> query_vec LIMIT 10` |

---

## B-Tree Indexes

The default and most versatile index type. Supports equality (`=`), range (`<`, `>`, `BETWEEN`), `ORDER BY`, `IS NULL`, and prefix matching (`LIKE 'abc%'`).

```sql
-- Simple B-tree (automatically created for PK and UNIQUE constraints)
CREATE INDEX idx_orders_created_at ON orders (created_at);
```

### Key Properties

- **Sorted structure:** Naturally supports `ORDER BY` without a separate sort step.
- **Prefix rule:** A composite B-tree on `(a, b, c)` can serve queries on `a`, `(a, b)`, or `(a, b, c)` — but NOT `b` alone or `(b, c)` (see Skip Scan exception below).
- **NULLs indexed:** B-tree indexes include NULL values and can serve `IS NULL` / `IS NOT NULL` queries.

---

## Skip Scan (PG18+)

PostgreSQL 18 introduces **Index Skip Scans**, which allow the planner to use a composite index even when the **leading column is NOT in the WHERE clause**.

```sql
-- Index on (category_id, created_at)
-- Pre-PG18: query on created_at alone cannot use this index
-- PG18+:   planner "skips" through distinct category_id values
SELECT * FROM products WHERE created_at > now() - interval '1 day';
```

**Design implication:** Place low-cardinality columns (status, region, tenant_id) as the leading column in composite indexes. Skip Scan is most effective when the leading column has few distinct values.

This reduces the total number of indexes needed — one composite index can serve both `WHERE category = ? AND created_at > ?` and `WHERE created_at > ?`.

---

## Composite Indexes

Multi-column indexes. Column order is critical.

### Column Ordering Rules

1. **Equality columns first:** Columns used with `=` in the WHERE clause go leftmost.
2. **Range/sort columns after:** Columns used with `>`, `<`, `BETWEEN`, or `ORDER BY` follow.
3. **Most selective first** (within equality columns): The column that filters the most rows goes first.

```sql
-- Query: WHERE workspace_id = ? AND status = 'active' ORDER BY created_at DESC
-- Optimal index:
CREATE INDEX idx_tasks_lookup
    ON tasks (workspace_id, status, created_at DESC);
```

### When NOT to Use Composite Indexes

- If the individual columns are queried independently in different queries, create separate single-column indexes instead.
- Every column added costs write amplification. Justify each column.

---

## Covering Indexes

Include non-key columns to enable **index-only scans** — the query is answered entirely from the index without visiting the heap (table).

```sql
CREATE INDEX idx_users_email_covering
    ON users (email) INCLUDE (name, avatar_url);

-- This query uses an index-only scan (no heap fetch):
SELECT name, avatar_url FROM users WHERE email = 'user@example.com';
```

### When to Use

- High-frequency read queries where the result set is small and specific.
- The included columns are read-often but not filtered/sorted on.

### Rules

- `INCLUDE` columns are **not part of the sort key** — they cannot be used for WHERE filtering or ORDER BY.
- `INCLUDE` columns **add storage** to every index entry.
- Only add columns that are actually needed by the covering query. Don't turn indexes into wide-row copies.

---

## Partial Indexes

Index only a subset of rows. Dramatically smaller and faster for filtered queries.

```sql
-- Only index non-deleted records (soft-delete pattern)
CREATE INDEX idx_users_active
    ON users (email)
    WHERE deleted_at IS NULL;

-- Only index pending orders (hot subset)
CREATE INDEX idx_orders_pending
    ON orders (user_id, created_at DESC)
    WHERE status = 'pending';
```

### Rules

- The query's WHERE clause must **match or imply** the index's WHERE predicate for the planner to use it.
- Partial unique indexes enforce uniqueness only over the included subset:

```sql
-- Unique email only among active users
CREATE UNIQUE INDEX uq_users_email_active
    ON users (LOWER(email))
    WHERE deleted_at IS NULL;
```

---

## Expression Indexes

Index on a computed expression rather than a raw column value.

```sql
-- Case-insensitive email lookup
CREATE UNIQUE INDEX uq_users_email_lower ON users (LOWER(email));

-- Date-only extraction from timestamp
CREATE INDEX idx_events_date ON events (date_trunc('day', created_at));
```

### Critical Rule

The expression in the WHERE clause must **exactly match** the indexed expression:

```sql
-- ✅ Uses the index
WHERE LOWER(email) = 'user@example.com'

-- ❌ Does NOT use the index (different expression)
WHERE email ILIKE 'user@example.com'
```

---

## GIN Indexes

**Generalized Inverted Index.** Optimized for types that contain multiple values (JSONB, arrays, tsvector, trigrams).

### JSONB Indexing

```sql
-- Default: supports all JSONB operators (@>, ?, ?|, ?&)
CREATE INDEX idx_profiles_data ON profiles USING GIN (data);

-- Path-ops: 3-4x smaller, supports only containment (@>)
CREATE INDEX idx_profiles_data_pathops
    ON profiles USING GIN (data jsonb_path_ops);
```

**Choose between them:**

| Operator Class | Supports | Size | Use When |
|---|---|---|---|
| `jsonb_ops` (default) | `@>`, `?`, `?|`, `?&` | Larger | You query key existence OR containment |
| `jsonb_path_ops` | `@>` only | 3-4x smaller | Containment queries dominate |

For full JSONB operator reference, hot-path extraction, and expression indexes on specific paths, see `jsonb.md`.

### Array Indexing

```sql
CREATE INDEX idx_articles_tags ON articles USING GIN (tags);
-- Fast: WHERE tags @> ARRAY['postgres']
-- Fast: WHERE tags && ARRAY['postgres', 'mysql']  (overlap)
```

### Full-Text Search

```sql
CREATE INDEX idx_articles_search ON articles USING GIN (search_vector);
-- Fast: WHERE search_vector @@ to_tsquery('english', 'database & design')
```

### Trigram Search (pg_trgm)

```sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE INDEX idx_users_name_trgm ON users USING GIN (name gin_trgm_ops);
-- Fast: WHERE name ILIKE '%john%'  (substring match)
-- Fast: WHERE name % 'johnn'       (fuzzy similarity)
```

### GIN Performance Notes

- GIN builds are slower than B-tree. Use `CREATE INDEX CONCURRENTLY` on live tables.
- GIN is optimized for reads, not writes. On write-heavy tables, consider `fastupdate = off` if query latency matters more than insert speed.

---

## GiST Indexes

**Generalized Search Tree.** Used for range types, geometric data, and exclusion constraints.

```sql
-- Range overlap prevention
CREATE TABLE reservations (
    room_id  BIGINT     NOT NULL,
    period   TSTZRANGE  NOT NULL,

    CONSTRAINT excl_reservations_no_overlap
        EXCLUDE USING GIST (room_id WITH =, period WITH &&)
);

-- For multi-type exclusion, enable btree_gist:
CREATE EXTENSION IF NOT EXISTS btree_gist;
```

- Required for `EXCLUDE` constraints with range operators.
- Used by PostGIS for spatial indexing.
- Generally less efficient than B-tree for simple equality/range — only use when operators require it.

---

## BRIN Indexes

**Block Range Index.** Extremely compact. Stores min/max values per block range rather than indexing every row.

```sql
CREATE INDEX idx_logs_created_at ON logs USING BRIN (created_at)
    WITH (pages_per_range = 128);
```

### When BRIN Works Well

- **Large tables** (millions to billions of rows).
- **Physical ordering correlates with indexed column** — append-only tables where rows are inserted in timestamp order.
- Works best after `CLUSTER` or when data is naturally ordered by insertion time.

### When BRIN Fails

- Random insert patterns — BRIN blocks span wide value ranges, defeating pruning.
- Small tables — overhead isn't worth it; use B-tree instead.

### Tuning `pages_per_range`

- **Lower value (32-64):** More precise, larger index, better selectivity.
- **Higher value (128-256):** Smaller index, less precise, works for coarse time-range queries.

---

## pgvector Indexes

### HNSW (Recommended Default)

Hierarchical Navigable Small World graph. High recall, handles incremental inserts, no periodic rebuild needed.

```sql
-- Cosine distance (most common for text embeddings)
CREATE INDEX idx_embeddings_hnsw
    ON embeddings USING hnsw (embedding vector_cosine_ops)
    WITH (m = 16, ef_construction = 64);
```

**Build Parameters (set once at index creation):**

| Parameter | Default | Guidance |
|---|---|---|
| `m` | 16 | Connections per node. Higher = better recall + more memory. Range: 16-64 |
| `ef_construction` | 64 | Build-time search width. Higher = better index quality + longer build. Range: 64-200 |

**Query-Time Parameters (set per session/transaction):**

```sql
-- Increase recall for this session
SET LOCAL hnsw.ef_search = 100;  -- default 40, range 40-200

-- Enable iterative scan for filtered queries (pgvector 0.8.0+)
SET LOCAL hnsw.iterative_scan = relaxed_order;
```

### IVFFlat (Large Static Datasets)

Inverted File with Flat quantization. Faster build, lower memory, but requires rebuild after significant data changes.

```sql
CREATE INDEX idx_embeddings_ivfflat
    ON embeddings USING ivfflat (embedding vector_cosine_ops)
    WITH (lists = 1000);
```

| Parameter | Guidance |
|---|---|
| `lists` | Number of clusters. Rule of thumb: `rows / 1000` for <1M rows, `sqrt(rows)` for larger |
| `probes` (query-time) | Number of clusters to scan. Default 1. Set via `SET ivfflat.probes = 10;` |

### HNSW vs IVFFlat Decision

| Factor | HNSW | IVFFlat |
|---|---|---|
| Incremental inserts | ✅ Handles gracefully | ❌ Requires rebuild |
| Query recall | Higher out-of-box | Requires probe tuning |
| Build time | Minutes to hours | Seconds to minutes |
| Memory | 2-5x raw data | Compact |
| Best for | Most production workloads | Very large (50M+), static datasets |

### DiskANN (Massive-Scale, Disk-Optimized)

For datasets exceeding available RAM, DiskANN (available via `pgvectorscale`) stores the graph on disk with an optimized I/O pattern.

- Best suited for self-hosted deployments with large NVMe storage.
- Designed for billion-scale vector search where HNSW's memory requirements become prohibitive.
- Currently less mature in the PostgreSQL ecosystem — prefer HNSW for most workloads.

### Distance Operators

| Operator | Distance Metric | Index Ops Class | Use When |
|---|---|---|---|
| `<->` | L2 (Euclidean) | `vector_l2_ops` | Geometric similarity |
| `<=>` | Cosine | `vector_cosine_ops` | Text/semantic embeddings (magnitude-invariant) |
| `<#>` | Inner Product (negative) | `vector_ip_ops` | Pre-normalized vectors, ranking |

### Operational Practices

- **Always `CREATE INDEX CONCURRENTLY`** on live tables.
- **Normalize vectors at ingestion** if using cosine distance — this allows using inner product (`<#>`) which is computationally cheaper.
- **Combine with metadata filters:** Add B-tree indexes on filter columns (`category`, `tenant_id`). Use `hnsw.iterative_scan` for filtered queries.
- **Monitor recall:** Periodically compare HNSW results against exact k-NN (`ORDER BY embedding <=> query LIMIT k` without index) on a sample.

---

## Index Maintenance

### Monitoring Unused Indexes

```sql
SELECT
    schemaname || '.' || relname AS table,
    indexrelname AS index,
    idx_scan AS scans,
    pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
ORDER BY pg_relation_size(indexrelid) DESC;
```

Drop indexes with zero scans after a full business cycle (at least one month of data).

### Index Bloat and Reindex

```sql
-- Online reindex (PG12+)
REINDEX INDEX CONCURRENTLY idx_orders_user_id;

-- Reindex entire table
REINDEX TABLE CONCURRENTLY orders;
```

### Concurrent Index Creation

```sql
-- Non-blocking index build (CANNOT run inside a transaction)
CREATE INDEX CONCURRENTLY idx_orders_status
    ON orders (status);
```

- If a concurrent build fails, it leaves an `INVALID` index. Check and drop it:

```sql
SELECT indexrelname, indisvalid
FROM pg_stat_user_indexes JOIN pg_index ON indexrelid = pg_stat_user_indexes.indexrelid
WHERE NOT indisvalid;
```

---

## Anti-Patterns

| Anti-Pattern | Problem | Fix |
|---|---|---|
| Indexing every column | Write amplification, storage waste | Index only queried columns |
| Duplicate indexes | Same key appears in multiple indexes | Audit with `pg_stat_user_indexes` |
| Function on indexed column in WHERE | `WHERE DATE(created_at) = ...` — index not used | Use range predicate or expression index |
| Wide composite indexes without justification | Each column costs write overhead | Justify each column in the composite |
| Blocking `CREATE INDEX` on live tables | Locks out all writes during build | Always use `CONCURRENTLY` |
| BRIN on randomly ordered data | Block ranges span entire value range | Use B-tree instead |
| Ignoring `INCLUDE` for high-freq queries | Heap fetches on every query | Add covering columns |
