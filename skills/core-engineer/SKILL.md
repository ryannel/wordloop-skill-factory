---
name: core-engineer
description: >
  Implement and review wordloop-core Go service changes using canonical docs
  and the current repository as the source of truth. Use for Go backend
  handlers, services, providers, domain models, migrations, telemetry,
  structured logging, dependency injection, tests, concurrency, or Core
  architecture. This skill is the execution router for Core work: it loads
  canonical docs selectively, preserves hexagonal boundaries, coordinates with
  API/database/real-time/testing/platform skills, and verifies changes against
  contracts and tests. Make sure to use this skill whenever the user works in
  wordloop-core or asks about Core Go backend behavior, even if they do not
  explicitly ask for a "core engineer."
---

# Core Engineer

Go backend execution router for `wordloop-core`. Durable engineering guidance lives in the docs site; this skill decides what to load, how to route the task, what repository facts to verify, and which safety gates apply.

## Operating Contract

1. Treat canonical docs and generated contracts as source-of-truth knowledge. Do not rely on skill-local architecture or Go reference material.
2. Inspect the current repository before naming packages, commands, import paths, schemas, or generated files.
3. Load the smallest docs set that explains the task. Add more context only when the task crosses a boundary.
4. Preserve Core's dependency direction and public contracts. Code implements OpenAPI, database migrations, event schemas, and documented architecture; it does not invent them.
5. Coordinate with adjacent skills when another skill owns the primary decision surface.

## Required First Checks

Before non-trivial Core implementation or review work:

- Check `services/wordloop-core/` package layout and nearby examples for the touched layer.
- Check `services/wordloop-core/go.mod` for Go and dependency versions before giving version-specific advice.
- Check relevant contracts before public behavior changes: OpenAPI for HTTP, `services/wordloop-core/db/schema.sql` plus `cmd/migrate` for schema, event references for async/real-time behavior.
- Prefer Wordloop docs MCP tools for docs context when available; otherwise read local MDX under `services/wordloop-docs/content/docs/`.

## Context Routing

Load only the rows needed for the user task.

| Task shape | Canonical context |
|---|---|
| Any non-trivial Core change | `learn/services/core/architecture`, `learn/services/core/implementation` |
| Go idioms, context propagation, interfaces, dependency injection, errors | `principles/stack/go-services` |
| Concurrency, goroutine lifecycle, errgroup, context cancellation, supervised goroutines | `principles/stack/go-services/concurrency` |
| Layer placement, new service/provider/domain boundary, dependency direction | `principles/system-design/hexagonal-architecture`, `learn/services/core/architecture` |
| HTTP endpoint, handler behavior, request/response mapping, generated clients | `learn/services/core/http-handlers`, `guides/add-api-endpoint`, `principles/system-design/api-design` |
| Database repository, SQL query, schema change, index, constraint, RLS, pgvector | `principles/stack/postgres`, `guides/migrate-schema`, `reference/database`, `services/wordloop-core/db/schema.sql`, `services/wordloop-core/cmd/migrate` |
| Observability, tracing, structured logging, metrics | `principles/quality/observability`, `learn/services/core/implementation` |
| Reliability, retries, timeouts, graceful shutdown, backpressure | `principles/quality/reliability`, `principles/stack/go-services` |
| Events, Pub/Sub, WebSocket, live synchronization, async integration | `principles/system-design/integration-patterns`, `principles/system-design/real-time`, event references |
| Tests, quality gates, coverage strategy, flake triage | `principles/stack/go-services/testing`, `principles/foundations/testing`, `guides/run-tests` |
| Local commands, generation, submodules, repo workflow | `reference/cli`, `guides/code-generation`, `start/first-contribution` |

## Skill Handoffs

Use the smallest collaborating set of skills.

- Use `api-architect` when endpoint shape, OpenAPI, error envelope, pagination, idempotency, SDK generation, webhooks, or versioning is the main question.
- Use `postgres-designer` when the task changes schema, migrations, indexes, query plans, constraints, RLS, retention, or vector/full-text search.
- Use `real-time-architect` when the task involves streaming, Pub/Sub, WebSockets, event schemas, replay, fan-out, idempotency, or source-aware updates.
- Use `test-architect` when the task is primarily test strategy, CI quality gates, contract tests, flake reduction, or validation design.
- Use `platform-engineer` when the task touches deployment, Cloud Run, Terraform, Docker, CI/CD, observability infrastructure, or local developer tooling.
- Keep `core-engineer` as lead when the work is mainly Go implementation inside `wordloop-core`.

## Execution Checklist

1. Identify the touched source-of-truth surface: domain behavior, HTTP contract, database schema, event contract, operational behavior, or tests.
2. Load the minimal routed docs and inspect nearby code before designing.
3. State any important inference when guidance comes from general Go knowledge rather than Wordloop docs or code.
4. Implement within the existing layer and package conventions; do not create new boundaries without evidence from docs or existing code.
5. Run targeted tests/checks when feasible. If not feasible, explain the blocker and name the exact command that should be run later.
6. Summarize docs consulted, files changed, verification performed, and residual risks.

## Safety Gates

- Do not invent package paths, command names, contract files, or service boundaries. Verify them in the repository first.
- Do not put business decisions in handlers, providers, middleware, migrations, or generated code when the current architecture expects them in domain/service code.
- Do not leak provider-specific types across layer boundaries.
- Do not log and return the same error at multiple layers. Prefer wrapping with context and logging at the boundary established by existing code.
- Do not launch untracked goroutines. Verify cancellation, error collection, and shutdown behavior for concurrent work.
- Do not change HTTP behavior without checking the OpenAPI source and generated-client workflow.
- Do not change database behavior without checking the target-state schema in `services/wordloop-core/db/schema.sql`, the `cmd/migrate` pg-schema-diff workflow, and `./dev db dry-run` output when feasible.
- Do not run generation manually outside documented `./dev gen` flows.

## Output Expectations

- Name the docs or source files that informed non-obvious decisions.
- Separate verified repository facts from recommendations or general Go inferences.
- Provide concrete verification commands and results.
- For code reviews, findings come first, ordered by severity, with file references and missing-test risks.
- For implementation work, keep the final summary focused on changed files, behavior, tests, and follow-up risks.
