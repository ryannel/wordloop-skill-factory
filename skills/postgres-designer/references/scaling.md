# Scaling Architecture Reference

## Table of Contents
- [Connection Pooling](#connection-pooling)
- [Read Replicas](#read-replicas)
- [Horizontal Sharding](#horizontal-sharding)
- [Background Job Patterns](#background-job-patterns)

---

## Connection Pooling

PostgreSQL's process-per-connection model is a scalability bottleneck. Each connection consumes 5-10MB of RAM.

### PgBouncer (Recommended)

Deploy PgBouncer in **transaction mode** between the application and PostgreSQL.

| Setting | Recommended | Why |
|---|---|---|
| Mode | `transaction` | Connections returned to pool after each transaction |
| `max_client_conn` | 1000+ | Handle many app connections |
| `default_pool_size` | 20-50 per database | Actual PG connections per pool |
| `reserve_pool_size` | 5 | Burst capacity |

### Pool Sizing Formula

```
pool_size = (num_cpu_cores * 2) + num_disk_spindles
```

For cloud NVMe: `pool_size ≈ CPU cores * 2 + 1`. Rarely exceeds 100-200 even for large deployments.

### What Breaks in Transaction Mode

- **Prepared statements:** Must use protocol-level prepared statements, not SQL `PREPARE`.
- **Session-level settings:** `SET` commands apply only within the transaction. Use `SET LOCAL` instead.
- **`LISTEN/NOTIFY`:** Requires session mode or a dedicated connection.
- **Temporary tables:** Dropped when the connection returns to the pool.

---

## Read Replicas

Scale read-heavy workloads by deploying **streaming replicas** and routing reads to them.

### Design Pattern

- Route `SELECT` / `GET` queries to read replicas.
- Reserve the primary node for writes (`INSERT`, `UPDATE`, `DELETE`).
- Application-level or proxy-level routing (e.g., PgBouncer with separate pools, or application middleware).

### Replication Lag Considerations

| Scenario | Tolerance | Routing |
|---|---|---|
| Dashboard analytics | Seconds to minutes | Always route to replica |
| User profile display after edit | Zero lag expected | Route to primary for read-after-write consistency |
| Real-time counters, balances | Zero lag | Primary only |

**Design implication:** If your application requires read-after-write consistency for specific entities, you need session-level routing or causal consistency markers — not just blanket replica routing.

---

## Horizontal Sharding

When a single primary node cannot handle the write throughput or data volume.

### Relational Sharding Pattern

Shard by a common key — typically `tenant_id` or `workspace_id` — to ensure that related data is **co-located on the same physical node**. This allows the shard to execute entire joins locally without cross-shard coordination.

```sql
-- Every tenant-scoped table includes the shard key
CREATE TABLE orders (
    id           UUID NOT NULL DEFAULT uuidv7(),
    workspace_id UUID NOT NULL,  -- shard key
    total        NUMERIC NOT NULL,
    -- ...
    CONSTRAINT pk_orders PRIMARY KEY (workspace_id, id)  -- shard key in PK
);
```

**Design implications for sharded schemas:**
- Every PK and unique constraint must include the shard key.
- Cross-shard JOINs are expensive or impossible — design entities to be shard-local.
- Foreign keys that cross shard boundaries are not enforceable — use application-level integrity.

### Sharding Technologies

| Tool | Approach | Best For |
|---|---|---|
| **Citus** (Microsoft) | Extension, transparent sharding | Multi-tenant SaaS, distributed tables |
| **Multigres** (Supabase) | Vitess-adapted proxy layer | Large-scale PostgreSQL clusters |
| **Neki** (PlanetScale) | Vitess-native horizontal sharding | Operational simplicity at massive scale |

---

## Background Job Patterns

Replace external job queues with PostgreSQL-native patterns for lightweight task processing.

### LISTEN/NOTIFY (Event Notification)

```sql
-- Publisher
NOTIFY new_task, '{"task_id": 42, "type": "email"}';

-- Subscriber (in application)
LISTEN new_task;
-- Application receives async notification
```

Lightweight pub/sub for triggering workers. Note: notifications are **not durable** — if no listener is connected, the message is lost. Use for cache invalidation, UI refresh triggers, not critical job processing.

### FOR UPDATE SKIP LOCKED (Durable Job Queue)

```sql
-- Worker claims the next unclaimed job atomically
WITH next_job AS (
    SELECT id, payload
    FROM job_queue
    WHERE status = 'pending'
    ORDER BY created_at
    LIMIT 1
    FOR UPDATE SKIP LOCKED
)
UPDATE job_queue
SET status = 'processing', claimed_at = now()
FROM next_job
WHERE job_queue.id = next_job.id
RETURNING job_queue.*;
```

Durable, ACID-compliant job queue without external infrastructure. Multiple workers can poll concurrently — `SKIP LOCKED` ensures no row contention.

### pg_cron (Scheduled Tasks)

```sql
-- Refresh materialized view every 15 minutes
SELECT cron.schedule('refresh_dashboard', '*/15 * * * *',
    $$REFRESH MATERIALIZED VIEW CONCURRENTLY mv_dashboard$$);

-- Partition maintenance: create next month's partition
SELECT cron.schedule('create_partitions', '0 0 25 * *',
    $$SELECT create_next_monthly_partition('events')$$);
```

Eliminates external cron dependencies for database-local tasks: materialized view refreshes, partition creation, old data cleanup, vacuum scheduling.
