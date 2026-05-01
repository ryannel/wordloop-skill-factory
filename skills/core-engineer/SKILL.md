---
name: core-engineer
description: >
  Implement and review wordloop-core Go service changes using canonical Go
  service docs, hexagonal architecture, context propagation, structured
  errors, observability, migrations, and contract-first API boundaries. Use
  for Go backend code, handlers, services, providers, domain models,
  migrations, telemetry, structured logging, dependency injection, tests,
  concurrency, or Core architecture. Make sure to use this skill whenever a
  user is working on the wordloop-core codebase, writing Go backend logic,
  fixing service bugs, or reviewing Go code, even if they don't explicitly
  ask for a "core engineer."
---

# Core Engineer

Go backend execution engineer for wordloop-core. This skill guides implementation within Wordloop's hexagonal architecture — strict layer boundaries, domain-driven modeling, contract-first APIs, and observable, resilient service behavior.

## Operating Contract

1. Preserve inward dependencies: domain knows nothing about infrastructure; services orchestrate domain logic; providers adapt external systems; entrypoints translate protocols.
2. Check canonical Core docs before changing architecture or public behavior.
3. Use contracts (OpenAPI) and migrations as source-of-truth surfaces. Code implements the contract, not the other way around.
4. Keep business logic out of transports and providers. If a handler contains domain logic, the boundary is wrong.

## Core Pillars

1. **Hexagonal Architecture** — The codebase is organized into concentric layers: domain (entities, value objects, repository interfaces), gateways (port definitions), services (orchestration), providers (adapters to external systems), and entrypoints (HTTP handlers, CLI). Dependencies always point inward. This separation means you can swap a Postgres provider for an in-memory one without touching business logic, and test services without spinning up infrastructure.

2. **Domain Integrity** — Business rules live in domain types, not scattered across handlers and providers. Value objects enforce constraints at construction. Repository interfaces are defined by the domain, implemented by providers. Errors are domain concepts with structured types, not stringly-typed messages. This makes the domain self-documenting and testable in isolation.

3. **Context Propagation** — Every operation carries a `context.Context` from entrypoint through service to provider. Context carries cancellation, deadlines, trace IDs, and request-scoped values. Never store contexts in structs or pass `context.Background()` when a real context is available. This is the foundation for tracing, timeouts, and graceful shutdown.

4. **Observable by Default** — Every meaningful operation emits structured logs (slog), traces (OpenTelemetry), and metrics. Observability is wired at the provider and entrypoint layers, not sprinkled ad-hoc through business logic. When something fails in production, the traces and logs should tell the story without requiring a code read.

5. **Safe Concurrency** — Go makes concurrency easy to start and hard to get right. Use `errgroup` for coordinated goroutines. Never launch unsupervised goroutines. Respect context cancellation in long-running operations. Prefer channels for communication and mutexes only when channel semantics don't fit.

## How to Use This Skill

Match the user's task to the smallest relevant reference set. Most tasks touch one or two references.

| Topic | Reference | Load When |
|-------|-----------|-----------|
| Architecture & Layers | `references/architecture.md` | Understanding the hexagonal layer structure, dependency rules, module organization. |
| Hexagonal Architecture | `references/hexagonal-architecture.md` | Deep dive into ports/adapters, dependency inversion, layer boundary decisions. |
| Domain Modeling | `references/domain-modeling.md` | Designing entities, value objects, repository interfaces, domain errors. |
| Service Layer | `references/service.md` | Service orchestration patterns, transaction boundaries, cross-cutting concerns. |
| Gateway Layer | `references/gateway.md` | Port definitions, gateway interfaces, adapter contracts. |
| Provider Layer | `references/provider.md` | Implementing adapters for databases, external APIs, message queues. |
| Entrypoint Layer | `references/entrypoint.md` | HTTP handlers, middleware, request/response mapping, validation. |
| Interfaces | `references/interfaces.md` | Go interface design, implicit satisfaction, small interfaces, testing seams. |
| Generics | `references/generics.md` | When and how to use Go generics effectively. |
| Go Idioms | `references/go-idioms.md` | Idiomatic Go patterns, error handling, naming, package design. |
| Modern Go | `references/modern-golang.md` | Recent Go features, iter package, structured logging, toolchain updates. |
| Concurrency | `references/concurrency.md` | Goroutines, errgroup, channels, context cancellation, mutex patterns. |
| Observability | `references/observability.md` | Structured logging with slog, OpenTelemetry tracing, metrics. |
| Observability & eBPF | `references/observability-and-ebpf.md` | Advanced observability, eBPF-based monitoring, resource tracking. |
| Resiliency & Lifecycle | `references/resiliency-and-lifecycle.md` | Circuit breaking, retries, graceful shutdown, health checks. |
| Testing & Quality | `references/testing-and-quality.md` | Test strategy, table-driven tests, testcontainers, fuzzing. |
| Testing | `references/testing.md` | Test patterns, mocking strategies, integration test setup. |
| Documentation | `references/documentation.md` | Code documentation patterns, godoc, package-level docs. |

## Required Wordloop Context

Read the relevant canonical docs before making non-trivial recommendations or code changes. Prefer the Wordloop docs MCP tools when available; otherwise read the local MDX files under `services/wordloop-docs/content/docs/`.

- `principles/stack/go-services` — Wordloop's Go conventions and technology choices.
- `principles/system-design/hexagonal-architecture` — The architectural pattern governing all services.
- `learn/services/core/architecture` — Core service structure, layers, and module map.
- `learn/services/core/implementation` — Data access, API patterns, and service wiring.

## Task Routing

- **HTTP/API work** → Read Core architecture and API Design principle. Check OpenAPI contract before modifying handlers.
- **Database work** → Read Postgres principle and migration guide. Load `references/provider.md` for repository implementation.
- **Domain modeling** → Load `references/domain-modeling.md`. Check existing domain types for conventions.
- **Telemetry/reliability work** → Load `references/observability.md` and `references/resiliency-and-lifecycle.md`.
- **Event/real-time work** → Read Integration Patterns and Real-Time principle.
- **Concurrency work** → Load `references/concurrency.md`. Verify context propagation and error handling.
- **New service/provider** → Load `references/architecture.md` and `references/hexagonal-architecture.md`. Verify layer placement.

## Safety Gates

- Do not invent package paths or service boundaries; inspect the repository structure first. The hex layers have specific package locations.
- Do not log and return the same error at multiple layers. Errors should be wrapped with context at each layer and logged once at the entrypoint.
- Do not launch unsupervised goroutines. Use `errgroup` or equivalent, and ensure context cancellation is respected.
- Do not put business logic in handlers or providers. If a handler is making domain decisions, extract to a service.
- Run Go tests or targeted checks where applicable and regenerate contracts after API changes.

## Hallucination Controls

Before presenting Go or Core guidance as factual:

- Check the actual package structure under `services/wordloop-core/` before naming packages or import paths.
- Check `go.mod` for Go version and dependency versions.
- Check existing code for patterns (error types, DI wiring, test setup) before proposing new conventions.
- Check the OpenAPI contract for endpoint shapes before modifying handlers.
- Label any recommendation based on general Go knowledge (rather than Wordloop-specific patterns) as an inference.

## Output Expectations

- Code changes respect hexagonal layer boundaries and dependency direction.
- New code includes appropriate observability (structured logs at provider/entrypoint, trace spans for significant operations).
- Error handling follows the wrap-and-propagate pattern with domain error types.
- Verification steps include specific test commands and files to check.
- Recommendations distinguish between Wordloop Core conventions and general Go best practices.

## Antipatterns

Reject these patterns:

- **Fat handlers** — HTTP handlers that contain business logic instead of delegating to services. Handlers translate HTTP; services make decisions.
- **Leaky providers** — Provider-specific types (SQL rows, SDK structs) leaking into service or domain layers. Providers map to domain types at the boundary.
- **Log-and-return** — Logging an error and returning it, causing the same error to appear multiple times in logs. Log once at the top; wrap with context at each layer.
- **Background goroutine fire-and-forget** — Launching goroutines without error collection, context awareness, or graceful shutdown. Use errgroup.
- **Interface pollution** — Defining interfaces before you have multiple implementations. In Go, interfaces are discovered by consumers, not declared by providers.
- **Stringly-typed errors** — Using `fmt.Errorf` for domain errors that callers need to inspect. Use typed error values or sentinel errors.
