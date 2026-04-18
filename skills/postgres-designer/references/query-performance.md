# Query Performance Reference

## Table of Contents
- [Performance Diagnosis Workflow](#performance-diagnosis-workflow)
- [EXPLAIN ANALYZE Guide](#explain-analyze-guide)
- [Common Performance Problems](#common-performance-problems)
- [pg_stat_statements](#pg_stat_statements)
- [Memory Tuning](#memory-tuning)
- [Vacuuming and Autovacuum](#vacuuming-and-autovacuum)
- [Partitioning](#partitioning)
- [Query Rewriting Patterns](#query-rewriting-patterns)
- [Server Configuration Baseline](#server-configuration-baseline)
- [Storage and I/O](#storage-and-io)

> For connection pooling, read replicas, horizontal sharding, and background job patterns, see `scaling.md`.

---

## Performance Diagnosis Workflow

1. **Identify hot queries** — Use `pg_stat_statements` to find queries with highest `total_exec_time`.
2. **Analyze the plan** — Run `EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)` on the hot query.
3. **Find the bottleneck** — Look for sequential scans, disk spills, sort overflows, and row estimate mismatches.
4. **Fix the bottleneck** — Add indexes, rewrite the query, tune `work_mem`, or update statistics.
5. **Verify the fix** — Re-run `EXPLAIN (ANALYZE, BUFFERS)` and compare buffer counts/timing.

---

## EXPLAIN ANALYZE Guide

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT o.id, o.total, u.name
FROM orders o
JOIN users u ON u.id = o.user_id
WHERE o.status = 'pending'
  AND o.created_at > now() - interval '7 days'
ORDER BY o.created_at DESC
LIMIT 20;
```

### Reading the Output

Each node in the plan shows:

| Field | Meaning |
|---|---|
| **Node Type** | The operation (Seq Scan, Index Scan, Hash Join, Sort, etc.) |
| **cost=0.00..123.45** | Estimated startup cost..total cost (arbitrary units) |
| **rows=1000** | Estimated number of rows |
| **actual time=0.5..12.3** | Real execution time in ms (startup..total) |
| **actual rows=950** | Actual number of rows returned |
| **Buffers: shared hit=100 read=20** | Cache hits vs disk reads (page count) |
| **Sort Method: quicksort Memory: 120kB** | Sort stayed in memory |
| **Sort Method: external merge Disk: 8192kB** | Sort spilled to disk — increase `work_mem` |

### Red Flags

| Symptom | Diagnosis | Fix |
|---|---|---|
| `Seq Scan` on large table with selective filter | Missing index | Add B-tree/GIN index matching the filter |
| `actual rows` ≫ `rows` (estimated) | Stale statistics | Run `ANALYZE table_name` |
| `Buffers: read` ≫ `Buffers: shared hit` | Cold cache or table too large for RAM | Check `shared_buffers`, add covering index |
| `external merge Disk` or `Hash Batches: N` | `work_mem` exceeded, spilling to disk | Increase `work_mem` for this session |
| `Nested Loop` with high outer row count | N+1 join pattern | Rewrite with `JOIN` or lateral join |
| `Bitmap Heap Scan` with `Recheck Cond` | Lossy bitmap — too many matching rows | More selective filter or partial index |

### Tips

- **Always use `BUFFERS`:** Timing is volatile (depends on load); buffer counts are stable and reveal actual I/O.
- **Run twice:** The first execution may be slow due to cold cache. The second shows warm-cache behavior. Compare both.
- **Use `FORMAT TEXT`** for human reading. Use `FORMAT JSON` for programmatic analysis.

---

## Common Performance Problems

### N+1 Queries

**Symptom:** Application makes one query to fetch parents, then N queries to fetch children.

**Fix:** Use a single join or `json_agg` to fetch everything in one round trip.

```sql
-- ❌ N+1: one query per order for items
SELECT * FROM orders WHERE user_id = 123;
-- then for each order:
SELECT * FROM order_items WHERE order_id = ?;

-- ✅ Single query with aggregation
SELECT o.*, json_agg(oi.*) AS items
FROM orders o
LEFT JOIN order_items oi ON oi.order_id = o.id
WHERE o.user_id = 123
GROUP BY o.id;
```

### OFFSET Pagination

**Symptom:** `OFFSET 10000 LIMIT 20` gets slower as page depth increases (scans and discards 10,000 rows).

**Fix:** Keyset (cursor) pagination.

```sql
-- ❌ OFFSET: O(N) performance
SELECT * FROM articles ORDER BY created_at DESC LIMIT 20 OFFSET 10000;

-- ✅ Keyset: O(1) performance — use last seen value as cursor
SELECT * FROM articles
WHERE created_at < '2026-01-15T12:00:00Z'  -- last seen timestamp
ORDER BY created_at DESC
LIMIT 20;
```

### COUNT(*) on Large Tables

**Symptom:** `SELECT COUNT(*) FROM big_table` takes seconds (full table scan).

**Fix:** Use approximate counts for UI display.

```sql
-- Approximate count (fast, from statistics)
SELECT reltuples::BIGINT FROM pg_class WHERE relname = 'big_table';

-- Or maintain a counter table for exact counts when needed
```

### Functions on Indexed Columns

```sql
-- ❌ Index on created_at NOT used (function wraps the column)
WHERE DATE(created_at) = '2026-01-15'

-- ✅ Range query uses the index
WHERE created_at >= '2026-01-15' AND created_at < '2026-01-16'
```

### NOT IN with NULLs

```sql
-- ❌ NOT IN returns no rows if subquery contains NULL
WHERE id NOT IN (SELECT parent_id FROM children)

-- ✅ NOT EXISTS handles NULLs correctly and performs better
WHERE NOT EXISTS (SELECT 1 FROM children c WHERE c.parent_id = t.id)
```

---

## pg_stat_statements

The essential extension for identifying which queries consume the most resources.

### Setup

```sql
-- postgresql.conf: shared_preload_libraries = 'pg_stat_statements'
-- Then restart PostgreSQL and run:
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
```

### Finding Top Queries

```sql
-- Top 10 by total execution time
SELECT
    queryid,
    calls,
    round(total_exec_time::numeric, 2) AS total_ms,
    round(mean_exec_time::numeric, 2) AS mean_ms,
    round((shared_blks_read + shared_blks_hit)::numeric, 0) AS total_blocks,
    rows,
    query
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
```

### Key Metrics

| Metric | What It Tells You |
|---|---|
| `total_exec_time` | Overall impact — optimize this first |
| `mean_exec_time` | Per-query latency — high values affect tail latency |
| `calls` | Frequency — N+1 queries show up as high call count |
| `shared_blks_read` | Disk I/O — high values suggest missing indexes or cold cache |
| `shared_blks_hit` | Cache hits — healthy when this is >95% of total blocks |
| `rows` | Rows returned — high per-call row count may indicate missing LIMIT |

### Reset

```sql
-- Reset after a release to measure impact of changes
SELECT pg_stat_statements_reset();
```

---


## Memory Tuning

### `work_mem`

Memory allocated **per operation** (sort, hash join, hash aggregate). A single complex query can use `work_mem` multiple times.

```sql
-- Conservative global default
-- work_mem = '16MB'

-- Increase for specific analytical sessions
SET LOCAL work_mem = '128MB';
```

| Setting | Risk |
|---|---|
| Too low (4MB default) | Sorts/hash joins spill to disk → slow queries |
| Too high (512MB+) | Concurrent queries exhaust RAM → OOM crashes |

**Safe estimation:**
```
work_mem = (Total RAM - shared_buffers - maintenance_work_mem) / (max_connections * 5)
```

The factor of 5 accounts for multiple operations per query.

### `shared_buffers`

PostgreSQL's internal cache. Set to **25% of system RAM**.

### `effective_cache_size`

Tells the planner how much data is likely cached by the OS page cache. Set to **50-75% of system RAM**. This does not allocate memory — it only affects query plan choices.

### `maintenance_work_mem`

Memory for `VACUUM`, `CREATE INDEX`, `ALTER TABLE`. Set higher than `work_mem` (e.g., 512MB-1GB) as these operations run infrequently but benefit from more memory.

### `hash_mem_multiplier` (PG13+)

Multiplier on `work_mem` for hash-based operations. Default 2.0. Consider increasing to 2.0-4.0 if hash joins frequently spill to disk.

---

## Vacuuming and Autovacuum

PostgreSQL MVCC creates dead tuples on every `UPDATE` and `DELETE`. `VACUUM` reclaims this space.

### Tuning Autovacuum for Write-Heavy Tables

```sql
ALTER TABLE high_churn_table SET (
    autovacuum_vacuum_threshold = 50,          -- rows changed before vacuum
    autovacuum_vacuum_scale_factor = 0.02,     -- 2% of table (default 20% is too high)
    autovacuum_analyze_threshold = 50,
    autovacuum_analyze_scale_factor = 0.01
);
```

### Key Monitoring Queries

```sql
-- Tables with most dead tuples
SELECT relname, n_dead_tup, n_live_tup,
       round(n_dead_tup::numeric / NULLIF(n_live_tup, 0) * 100, 1) AS dead_pct,
       last_autovacuum
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY n_dead_tup DESC;
```

### Preventing Bloat

- **Use `fillfactor=90`** on update-heavy tables — leaves room for HOT (Heap-Only Tuple) updates.
- **Avoid updating indexed columns** — HOT updates only work when no indexed column changes.
- **Separate hot/cold columns** — put frequently updated columns in a separate 1:1 table.

---

## Partitioning

Use for very large tables (>100M rows) where queries consistently filter on the partition key.

### Declarative Partitioning (PG10+)

```sql
-- Range partition by time
CREATE TABLE events (
    id         BIGINT GENERATED ALWAYS AS IDENTITY,
    event_type TEXT        NOT NULL,
    payload    JSONB,
    created_at TIMESTAMPTZ NOT NULL,

    CONSTRAINT pk_events PRIMARY KEY (id, created_at)  -- must include partition key
) PARTITION BY RANGE (created_at);

CREATE TABLE events_2026_01 PARTITION OF events
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
CREATE TABLE events_2026_02 PARTITION OF events
    FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');

-- Default partition catches rows that don't match any partition
CREATE TABLE events_default PARTITION OF events DEFAULT;
```

### Partition Key Rules

- **Include partition key in PK/UNIQUE:** Partitioned tables require the partition key in all unique constraints (this is a PostgreSQL requirement, not a convention).
- **Keep predicates sargable:** Use direct comparisons on the partition key. Avoid wrapping it in functions.
- **Monitor the DEFAULT partition:** If it grows, you're missing partition definitions.

### Partition Strategies

| Strategy | Best For | Example |
|---|---|---|
| `RANGE` | Time-series, date-ordered data | `PARTITION BY RANGE (created_at)` |
| `LIST` | Discrete categories, multi-tenant | `PARTITION BY LIST (region)` |
| `HASH` | Even distribution, no natural key | `PARTITION BY HASH (user_id)` |

### Data Retention via Partition Drop

```sql
-- Instant: drops entire partition at filesystem level
ALTER TABLE events DETACH PARTITION events_2025_01;
DROP TABLE events_2025_01;
-- No VACUUM overhead, no WAL generation for deletion
```

### pg_partman (Automated Partition Management)

For automated partition creation and retention:
- Automatically creates future partitions on a schedule
- Drops or detaches old partitions according to retention policy
- Runs via `pg_cron` or background worker

### TimescaleDB (Time-Series)

For metrics, logs, and IoT data, TimescaleDB transforms standard tables into **hypertables** — automatically partitioned into time-based chunks.

```sql
SELECT create_hypertable('metrics', 'time');
```

- **Automatic chunking:** Data is partitioned transparently by time interval.
- **Compression:** Columnar compression on old chunks (90%+ space savings). Reduces multi-TB tables to manageable sizes.
- **Continuous aggregates:** Pre-computed rollups that auto-refresh on new data.
- **Chunk-level retention:** Drop old chunks at the filesystem level — instant, no WAL generation, no `VACUUM` overhead. Far cheaper than `DELETE FROM ... WHERE time < cutoff`.

---

## Query Rewriting Patterns

### Lateral Joins (Replace Correlated Subqueries)

```sql
-- ❌ Correlated subquery (executed once per outer row)
SELECT u.*, (SELECT COUNT(*) FROM orders o WHERE o.user_id = u.id) AS order_count
FROM users u;

-- ✅ Lateral join (optimized by planner)
SELECT u.*, o.order_count
FROM users u
LEFT JOIN LATERAL (
    SELECT COUNT(*) AS order_count FROM orders WHERE user_id = u.id
) o ON true;
```

### CTEs for Readability (PG12+ Inlined)

Since PG12, CTEs are **inlined by default** (no longer optimization fences). Use freely for readability:

```sql
WITH active_orders AS (
    SELECT * FROM orders WHERE status = 'active'
)
SELECT u.name, COUNT(ao.id)
FROM users u
JOIN active_orders ao ON ao.user_id = u.id
GROUP BY u.name;
```

To force materialization (old behavior): `WITH active_orders AS MATERIALIZED (...)`.

### Batch Fetching with `= ANY()`

```sql
-- ❌ N separate queries
SELECT * FROM users WHERE id = 1;
SELECT * FROM users WHERE id = 2;
-- ...

-- ✅ Single batch query
SELECT * FROM users WHERE id = ANY(ARRAY[1, 2, 3, 4, 5]);
```

---

## Server Configuration Baseline

| Parameter | Recommended | Impact |
|---|---|---|
| `shared_buffers` | 25% of system RAM | Primary internal cache |
| `effective_cache_size` | 50-75% of system RAM | Planner cost estimates |
| `work_mem` | 16MB-256MB (default conservative, increase per-session) | Per-operation sort/hash memory |
| `maintenance_work_mem` | 512MB-1GB | VACUUM, CREATE INDEX speed |
| `max_wal_size` | 16GB-32GB | Prevents checkpoint spikes |
| `random_page_cost` | 1.1 (for SSDs) | Favors index scans over seq scans |
| `effective_io_concurrency` | 100-300 (for SSDs) | Concurrent read-ahead hints |
| `io_method` | `io_uring` (PG18+, Linux) | Async I/O for cold reads |
| `default_statistics_target` | 100-500 for important columns | Planner accuracy vs ANALYZE time |

### Per-Table Tuning

```sql
-- Write-heavy table: lower vacuum thresholds
ALTER TABLE high_churn SET (autovacuum_vacuum_scale_factor = 0.02);

-- Update-heavy table: leave room for HOT updates
ALTER TABLE frequently_updated SET (fillfactor = 90);
```

---

## Storage and I/O

### Async I/O (PG18+)

PostgreSQL 18's native async I/O subsystem allows concurrent disk reads, providing 2-3x speedup for cold-cache workloads on cloud storage.

| Setting | Recommended |
|---|---|
| `io_method` | `io_uring` (Linux 5.1+) or `worker` (other platforms) |
| `io_workers` | 1/4 of total CPU threads |
| `effective_io_concurrency` | 100-300 for NVMe/SSDs |

### Storage Best Practices

- **Filesystem:** XFS or ext4 with `noatime` mount flag.
- **Storage:** NVMe SSDs for data and WAL.
- **WAL separation:** Place WAL on a separate physical disk to prevent I/O contention.
- **TOAST tuning:** For wide tables, adjust `toast_tuple_target` to control inline vs out-of-line storage threshold.
