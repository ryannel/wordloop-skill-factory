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
- `reference/database` — Generated database reference for the current schema.
- `guides/migrate-schema` — Wordloop Core target-state schema workflow, tooling, and safety checklist.
- `services/wordloop-core/db/schema.sql` — Canonical source of truth for the Core Postgres schema.
- `services/wordloop-core/cmd/migrate` — `pg-schema-diff` migrator used by `./dev db dry-run` and `./dev db migrate`.

## Task Routing

- **New table/entity** → Read Postgres principle and migration guide. Load `references/table-design.md` and `references/types.md`. Start from access patterns.
- **Performance issue** → Run EXPLAIN ANALYZE on the problem query. Load `references/query-performance.md` and `references/indexes.md`.
- **Index changes** → Load `references/indexes.md`. Verify the index will be used with EXPLAIN ANALYZE before adding.
- **JSONB/vector work** → Load `references/jsonb.md`. Read Postgres principle for guidance on when to use JSONB vs normalized columns.
- **Schema change authoring** → Load `references/migrations.md`, then verify the target-state change in `services/wordloop-core/db/schema.sql` and review `./dev db dry-run` hazards for zero-downtime deployment risk.
- **Privacy/retention work** → Load `references/security.md`. Read Privacy principle and active design docs.
- **Scaling decisions** → Load `references/scaling.md`. Check current table sizes and growth projections.
- **TDD schema document** → Follow the TDD Schema Documentation rules below. Scaffold with `./dev new schema <bet-slug>`.

## TDD Schema Documentation

When writing a schema document for a bet's TDD (`tdd/schemas/postgres.mdx`), the document describes **target state** — how the schema works at completion. It is not a diff, gap analysis, or migration plan.

**Structure:** One section per domain area (e.g. Recording Lifecycle, Speaker Identification). Each section contains the full `CREATE TABLE` statement(s) for that domain's tables, including inline SQL comments that explain *why* design decisions were made. End with an Index Design table that justifies every non-obvious index.

**Rules:**

1. **Target state only.** Never use "New Table:", "Modified Table:", or "Dropped:" headings. Never write `-- changed: was X`, `-- ... existing columns ...`, or any diff notation. A reader unfamiliar with history should read this as a coherent description of how things work, not as a record of what changed.

2. **Full `CREATE TABLE` for every table shown.** No partial snippets. If a table is relevant to the domain, show every column in its final form. Inline SQL comments explain rationale; they do not annotate history.

3. **Comments explain WHY, not WHAT.** Column names already say what. Comments are for non-obvious constraints, invariants, trade-offs, and design choices that would surprise a future reader. Example: `-- REAL not DECIMAL: ML scores are approximate floats` is useful. `-- score column` is not.

4. **State machines and ER diagrams use the `<Mermaid>` component.** Fumadocs does not render fenced ` ```mermaid ``` ` blocks. Use `<Mermaid chart={`...`} />` with `import { Mermaid } from '@/components/Mermaid'` after the frontmatter. Every table with a meaningful lifecycle gets a `stateDiagram-v2`. The overview section gets an `erDiagram`.

5. **Index design is explicit and justified.** Every index appears in both the `CREATE TABLE` block (inline) and the Index Design table at the end of the document. The Index Design table states the query the index serves, not just the columns.

6. **Scaffold with `./dev new schema <bet-slug>`.** This creates `tdd/schemas/postgres.mdx`, a `tdd/schemas/meta.json`, and registers `schemas` in `tdd/meta.json`. The template at `services/wordloop-docs/content/docs/work/_template/.scaffolds/schema.mdx` provides the correct starting structure.

## Safety Gates

- Do not design schema without known access patterns. A table designed without knowing how it will be queried is a performance risk.
- Do not propose blocking or destructive migrations without expand-contract reasoning. Locking a large table during deployment causes downtime.
- Do not hand-edit the generated DB reference as the source fix. Update `services/wordloop-core/db/schema.sql`, review `./dev db dry-run`, apply with `./dev db migrate`, then refresh reference docs/snapshots.
- Do not add indexes without verifying they will be used. EXPLAIN ANALYZE is the validation, not intuition.
- Do not skip foreign key constraints for "performance" without measuring the actual cost and documenting the tradeoff.
- Run migration and test commands appropriate to the service.

## Hallucination Controls

Before presenting database guidance as factual:

- Check the generated database reference for current schema, table names, and column types.
- Check `services/wordloop-core/db/schema.sql` and `services/wordloop-core/cmd/migrate` for the current target-state workflow before discussing migrations.
- Check PostgreSQL version compatibility for features (e.g., MERGE, JSON path queries).
- Run EXPLAIN ANALYZE on actual data before recommending index or query changes.
- Label any recommendation based on general PostgreSQL knowledge (rather than Wordloop-specific schema) as an inference.

## Output Expectations

- Schema changes include the target-state SQL diff, `./dev db dry-run` plan/hazards, and a safety assessment (locking behavior, data volume impact, rollback/forward-fix plan).
- New tables include the access patterns they serve and the indexes they need.
- Performance recommendations include EXPLAIN ANALYZE output before and after.
- Verification steps include `./dev db dry-run`, `./dev db migrate`, and test scenarios.
- Recommendations distinguish between Wordloop database conventions and general PostgreSQL best practices.

## Antipatterns

Reject these patterns:

- **Schema without access patterns** — Designing tables from an entity model without knowing the query patterns. Beautiful normalization means nothing if every read requires five joins.
- **Destructive migration** — Dropping columns, renaming tables, or changing types without the expand-contract pattern. These lock tables and break running code.
- **Index speculation** — Adding indexes "just in case" without query evidence. Each index has write and storage costs.
- **JSONB as schema escape** — Using JSONB columns to avoid schema design decisions. JSONB is appropriate for genuinely dynamic data, not for avoiding migrations.
- **Application-only constraints** — Enforcing uniqueness, nullability, or referential integrity only in application code. The database should enforce invariants that matter.
- **Manual reference as source** — Treating database reference docs as the schema source. The source is `services/wordloop-core/db/schema.sql`; docs must be refreshed from it.
