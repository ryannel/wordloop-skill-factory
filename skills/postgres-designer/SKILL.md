---
name: postgres-designer
description: >
  Design and review PostgreSQL schemas, migrations, indexes, constraints,
  query plans, RLS, JSONB usage, and pgvector patterns using canonical
  Wordloop data docs. Use for schema modeling, migrations, table design,
  column types, UUID strategies, indexes, EXPLAIN plans, query performance,
  vector search, full-text search, partitioning, retention, privacy, row-
  level security, zero-downtime migrations, or database drift. Make sure to
  use this skill whenever a user is working on database schema changes,
  writing migrations, debugging slow queries, or designing data models, even
  if they don't explicitly ask for a "postgres designer."
---

# Postgres Designer

PostgreSQL execution designer for Wordloop data changes. This skill guides schema design, migration safety, index strategy, and query performance — starting from access patterns, growth projections, and data privacy requirements.

## Operating Contract

1. Model invariants in the database where practical. Constraints, foreign keys, and check expressions catch bugs that application code misses.
2. Start from access patterns, growth projections, tenancy model, privacy requirements, and migration safety before designing schema.
3. Treat migrations and the generated database reference as source-of-truth surfaces. Schema changes flow through migrations, not ad-hoc DDL.
4. Keep schema guidance in docs and use this skill for design workflow and verification.

## Core Pillars

1. **Access-Pattern-First Design** — Schema design starts with the queries, not the entities. Know the read/write ratio, expected row counts, hot paths, and join patterns before choosing types, indexes, and normalization level. A beautifully normalized schema that requires five-table joins on every page load is a performance liability, not good design.

2. **Migration Safety** — Every schema change must be deployable without downtime. Use the expand-contract pattern: add the new column/table, migrate data, switch reads, drop the old structure. Never run `ALTER TABLE ... ADD COLUMN NOT NULL` without a default on a large table. Never drop a column that active code still references. Migration safety is more important than schema elegance.

3. **Database-Enforced Invariants** — The database outlives the application code. Constraints (NOT NULL, UNIQUE, CHECK, FOREIGN KEY) encode business rules that survive code rewrites, API changes, and team turnover. If an invariant matters, model it in the schema. Application-only validation is a suggestion; a database constraint is a guarantee.

4. **Index Strategy** — Indexes are not free. Each index slows writes, consumes storage, and needs maintenance. Design indexes for specific query patterns, not "just in case." Use EXPLAIN ANALYZE to verify indexes are actually used. Understand when B-Tree, GIN, GiST, BRIN, or partial indexes are appropriate. An unused index is pure cost.

5. **Data Privacy by Design** — Row-level security, column encryption, retention policies, and audit trails are schema-level concerns, not afterthoughts. Design for data isolation from the start. Adding RLS to a mature schema is significantly harder than building it in from day one.

## How to Use This Skill

Match the user's task to the smallest relevant reference set. Most tasks touch one or two references.

| Topic | Reference | Load When |
|-------|-----------|-----------|
| Table Design | `references/table-design.md` | Designing new tables, choosing normalization level, entity relationships. |
| Data Types | `references/types.md` | Choosing column types, UUID strategies (v4 vs v7), timestamp handling, enums. |
| Constraints | `references/constraints.md` | Defining NOT NULL, UNIQUE, CHECK, FK constraints, exclusion constraints. |
| Indexes | `references/indexes.md` | Index type selection (B-Tree, GIN, GiST, BRIN), partial indexes, covering indexes. |
| Migrations | `references/migrations.md` | Zero-downtime migrations, expand-contract pattern, migration tooling. |
| Query Performance | `references/query-performance.md` | EXPLAIN ANALYZE, query optimization, common performance patterns. |
| JSONB | `references/jsonb.md` | JSONB column design, indexing strategies, when to use vs normalize. |
| Security | `references/security.md` | Row-level security, column encryption, audit trails, access control. |
| Scaling | `references/scaling.md` | Partitioning, sharding strategies, connection pooling, read replicas. |

## Required Wordloop Context

Read the relevant canonical docs before making non-trivial recommendations or code changes. Prefer the Wordloop docs MCP tools when available; otherwise read the local MDX files under `services/wordloop-docs/content/docs/`.

- `principles/stack/postgres` — Wordloop's PostgreSQL conventions and design philosophy.
- `reference/database` — Generated database reference (source of truth for current schema).
- `guides/migrate-schema` — Migration workflow, tooling, and safety checklist.

## Task Routing

- **New table/entity** → Read Postgres principle and migration guide. Load `references/table-design.md` and `references/types.md`. Start from access patterns.
- **Performance issue** → Run EXPLAIN ANALYZE on the problem query. Load `references/query-performance.md` and `references/indexes.md`.
- **Index changes** → Load `references/indexes.md`. Verify the index will be used with EXPLAIN ANALYZE before adding.
- **JSONB/vector work** → Load `references/jsonb.md`. Read Postgres principle for guidance on when to use JSONB vs normalized columns.
- **Migration authoring** → Load `references/migrations.md`. Verify the migration is safe for zero-downtime deployment.
- **Privacy/retention work** → Load `references/security.md`. Read Privacy principle and active design docs.
- **Scaling decisions** → Load `references/scaling.md`. Check current table sizes and growth projections.

## Safety Gates

- Do not design schema without known access patterns. A table designed without knowing how it will be queried is a performance risk.
- Do not propose blocking or destructive migrations without expand-contract reasoning. Locking a large table during deployment causes downtime.
- Do not hand-edit the generated DB reference as the source fix. Update the migration and regenerate.
- Do not add indexes without verifying they will be used. EXPLAIN ANALYZE is the validation, not intuition.
- Do not skip foreign key constraints for "performance" without measuring the actual cost and documenting the tradeoff.
- Run migration and test commands appropriate to the service.

## Hallucination Controls

Before presenting database guidance as factual:

- Check the generated database reference for current schema, table names, and column types.
- Check existing migrations for naming conventions, ordering, and patterns.
- Check PostgreSQL version compatibility for features (e.g., MERGE, JSON path queries).
- Run EXPLAIN ANALYZE on actual data before recommending index or query changes.
- Label any recommendation based on general PostgreSQL knowledge (rather than Wordloop-specific schema) as an inference.

## Output Expectations

- Schema changes include the migration SQL and a safety assessment (locking behavior, data volume impact, rollback plan).
- New tables include the access patterns they serve and the indexes they need.
- Performance recommendations include EXPLAIN ANALYZE output before and after.
- Verification steps include specific migration commands and test scenarios.
- Recommendations distinguish between Wordloop database conventions and general PostgreSQL best practices.

## Antipatterns

Reject these patterns:

- **Schema without access patterns** — Designing tables from an entity model without knowing the query patterns. Beautiful normalization means nothing if every read requires five joins.
- **Destructive migration** — Dropping columns, renaming tables, or changing types without the expand-contract pattern. These lock tables and break running code.
- **Index speculation** — Adding indexes "just in case" without query evidence. Each index has write and storage costs.
- **JSONB as schema escape** — Using JSONB columns to avoid schema design decisions. JSONB is appropriate for genuinely dynamic data, not for avoiding migrations.
- **Application-only constraints** — Enforcing uniqueness, nullability, or referential integrity only in application code. The database should enforce invariants that matter.
- **Manual reference editing** — Hand-writing database reference docs that should be generated from the actual schema.
