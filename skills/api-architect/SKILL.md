---
name: api-architect
description: >
  Design and review Wordloop API and event contracts using canonical docs,
  OpenAPI/AsyncAPI sources, agent-readable schemas, and safe lifecycle
  governance. Use when the user mentions REST APIs, OpenAPI, AsyncAPI,
  endpoint design, event schemas, versioning, pagination, idempotency,
  error envelopes, API security, SDK generation, API docs, agent-ready
  interfaces, webhook design, API gateways, rate limiting, caching strategy,
  content negotiation, or contract-first development. Make sure to use this
  skill whenever a user is working on API contracts, HTTP endpoint shapes,
  event payloads, or integration boundaries, even if they don't explicitly
  ask for an "API architect."
---

# API Architect

API execution architect for Wordloop contracts. This skill guides contract-first design of REST, WebSocket, and Pub/Sub interfaces with a focus on agent-readability, lifecycle safety, and consistency across Wordloop's integration surface.

## Operating Contract

1. Start from the contract source of truth (OpenAPI/AsyncAPI specs), not route-code guesses. Contracts define the promise; code implements it.
2. Keep REST, WebSocket, and Pub/Sub designs consistent with Wordloop API principles — shared error envelopes, consistent naming, stable pagination.
3. Treat agent-readability as part of API quality: descriptions, examples, constraints, stable errors, and idempotency enable both human developers and AI agents to consume APIs reliably.
4. Route durable policy to docs; keep this skill focused on contract workflow and verification.

## Core Pillars

1. **Contract-First Development** — The OpenAPI/AsyncAPI spec is the single source of truth for every endpoint and event. Design the contract before writing handler code. Generated clients, docs, and SDK references flow from the contract, not the reverse. This prevents drift between what the API promises and what it delivers.

2. **Resource-Oriented Design** — Model APIs around domain resources, not RPC actions. Resources have stable identifiers, predictable CRUD semantics, and consistent URL structures. This makes APIs discoverable for both humans and agents and reduces the cognitive load of learning new endpoints.

3. **Lifecycle Safety** — Every API change is either backwards-compatible or follows an explicit versioning and deprecation strategy. Breaking changes go through expand-contract migration. Sunset headers, deprecation notices, and version negotiation protect consumers from surprise breakage.

4. **Agent-Readable Interfaces** — APIs serve two audiences: human developers reading docs and AI agents consuming structured responses. Rich descriptions, typed examples, deterministic error codes, idempotency keys, and machine-parseable pagination make APIs reliable for automated consumption.

5. **Defensive Boundaries** — Every external-facing endpoint validates input, enforces rate limits, requires authentication, and returns structured errors. Treat service-to-service boundaries with similar rigor — internal APIs accumulate consumers quickly and become hard to change.

## How to Use This Skill

Match the user's task to the smallest relevant reference set. Most tasks touch one or two references.

| Topic | Reference | Load When |
|-------|-----------|-----------|
| Resource & URI Design | `references/resource-design.md` | Designing new endpoints, naming resources, structuring URL hierarchies. |
| HTTP Semantics | `references/http-semantics.md` | Choosing HTTP methods, status codes, content negotiation, conditional requests. |
| OpenAPI Specification | `references/openapi-spec.md` | Writing or reviewing OpenAPI specs, schema design, component reuse. |
| Pagination & Filtering | `references/pagination-filtering.md` | Designing list endpoints, cursor vs offset pagination, filter/sort query params. |
| Error Handling | `references/error-handling.md` | Designing error envelopes, RFC 9457 Problem Details, error code taxonomies. |
| Versioning & Lifecycle | `references/versioning-lifecycle.md` | API versioning strategy, deprecation, sunset headers, breaking change policy. |
| Security | `references/security.md` | Auth patterns, OAuth 2.1, API keys, rate limiting, CORS, input validation. |
| Webhooks | `references/webhooks.md` | Webhook contracts, delivery guarantees, retry policies, payload signing. |
| Async & Event Patterns | `references/async-and-event-patterns.md` | AsyncAPI contracts, Pub/Sub event design, event naming, payload schemas. |
| API Topologies | `references/api-topologies.md` | Gateway patterns, BFF, service mesh, multi-tier routing, load balancing. |
| AI Agent Readiness | `references/ai-agent-readiness.md` | Making APIs consumable by AI agents, description quality, deterministic responses. |
| Advanced Patterns | `references/advanced-patterns.md` | HATEOAS, bulk operations, long-running operations, ETags, content negotiation. |

## Required Wordloop Context

Read the relevant canonical docs before making non-trivial recommendations or code changes. Prefer the Wordloop docs MCP tools when available; otherwise read the local MDX files under `services/wordloop-docs/content/docs/`.

- `principles/system-design/api-design` — Wordloop's API design philosophy and standards.
- `principles/system-design/integration-patterns` — How services communicate across boundaries.
- `principles/ai-native/agent-native-systems` — Agent-readability requirements for all interfaces.
- `reference/api/core` — Generated Core API reference (source of truth for existing endpoints).
- `reference/events/core-ws` — WebSocket event reference.
- `reference/events/core-pubsub` — Pub/Sub event reference.

## Task Routing

- **New REST endpoint** → Read API Design principle and Core API reference. Design the contract first, then implement the handler.
- **WebSocket/Pub/Sub work** → Read Integration Patterns plus the relevant event reference. Check AsyncAPI specs.
- **SDK/docs generation** → Read code-generation guide. Verify generated outputs match the contract.
- **Error handling changes** → Load `references/error-handling.md`. Ensure RFC 9457 compliance and consistency with the shared error envelope.
- **Pagination/filtering** → Load `references/pagination-filtering.md`. Check consistency with existing list endpoints.
- **Security-sensitive changes** → Load `references/security.md`. Include Security and Privacy principles in review.
- **Versioning decisions** → Load `references/versioning-lifecycle.md`. Assess breaking change impact and migration plan.

## Safety Gates

- Do not invent endpoint paths, event names, or fields without checking OpenAPI/AsyncAPI specs or source code. Guessing at API shapes leads to contract drift.
- Do not manually edit generated reference docs; update the source contract and regenerate. Generated docs are downstream artifacts.
- Run `./dev gen api` or the relevant generation target when contracts change.
- Run `./dev docs health` after docs or contract reference updates.
- Do not introduce breaking changes without an expand-contract migration plan and consumer impact assessment.

## Hallucination Controls

Before presenting API guidance as factual:

- Check OpenAPI/AsyncAPI specs for endpoint paths, methods, status codes, and field names.
- Check source code for actual handler implementations when specs are ambiguous.
- Check `./dev gen --help` for available generation commands rather than guessing.
- Check existing endpoints for naming and pagination conventions before proposing new patterns.
- Label any recommendation not grounded in specs or code as an inference.

## Output Expectations

- Contract changes include both the OpenAPI/AsyncAPI diff and the implementation plan.
- New endpoints follow existing naming, error, and pagination conventions unless the deviation is justified.
- Generated references, SDK clients, and docs are regenerated after contract changes.
- Verification steps include specific commands to run and files to check.
- Recommendations distinguish between Wordloop conventions, general best practices, and inferences.

## Antipatterns

Reject these patterns:

- **Code-first API design** — Writing handlers before the contract leads to accidental API surfaces that are hard to change later.
- **Snowflake error formats** — Each endpoint inventing its own error structure instead of using the shared error envelope.
- **Undocumented pagination** — List endpoints that return unbounded results or use inconsistent cursor/offset schemes.
- **Version in URL without strategy** — Adding `/v2/` without a deprecation timeline or migration plan for `/v1/`.
- **Auth afterthought** — Designing the happy path first and bolting on security later, leading to inconsistent enforcement.
- **Manual reference editing** — Hand-writing API reference tables that should be generated from contracts.
