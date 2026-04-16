# Architectural Blueprints: Choosing Your Testing Shape

This module defines the testing architecture shapes and when to apply each. The shape of your test suite must match the technical complexity and risk profile of your system — there is no universal "testing pyramid" that fits all contexts.

---

## Table of Contents

1. [Why Shape Matters](#why-shape-matters)
2. [The Testing Honeycomb (Microservices)](#the-testing-honeycomb-microservices)
3. [The Testing Trophy (Web Applications)](#the-testing-trophy-web-applications)
4. [The Testing Pyramid (Legacy/Monoliths)](#the-testing-pyramid-legacymonoliths)
5. [Shape Selection Decision Matrix](#shape-selection-decision-matrix)
6. [Common Shape Antipatterns](#common-shape-antipatterns)

---

## Why Shape Matters

The "shape" of a test suite describes the distribution of investment across test types. Getting this wrong creates one of two failure modes:

- **Too bottom-heavy (excessive unit tests):** Tests are micro-scoped, tightly coupled to implementation details, and break on every refactor. High maintenance cost, low confidence in system behavior.
- **Too top-heavy (excessive E2E tests):** Tests are slow, flaky, expensive, and provide feedback too late. Pipeline times balloon, developers stop trusting the suite.

The right shape depends on the system's architecture, dependency model, and risk profile.

---

## The Testing Honeycomb (Microservices)

**The recommended default for microservice architectures.**

```
          ┌─────┐
         │ E2E │        ← Few, targeted
        ┌┘     └┐
       │ System │       ← System Test: cross-service
      ┌┘ Tests  └┐
     │           │
    ┌┘  Service  └┐     ← Service Test: THE CORE ← Maximum investment
   │    Tests     │
    └┐           ┌┘
     │   Unit   │       ← Rare, surgical
      └─────────┘
```

### Principle

**Integration/Service Tests are the "minimal fundamental unit."** Unit testing within a microservice is rare and reserved exclusively for isolating complex algorithmic logic.

### Rationale

In a microservice, the interesting behavior happens at boundaries: HTTP handling, database interaction, message publishing, external API calls. These are precisely the things that "solitary" unit tests mock away. A Service Test exercises all of these real boundaries while remaining hermetic and fast.

### When Unit Tests ARE Appropriate in the Honeycomb

- **Complex algorithms:** Sorting, scoring, ranking, mathematical calculations where correctness depends on precise logic.
- **Pure data transformations:** Converting between formats, parsing, serialization utilities.
- **State machines:** Complex state transition logic where exhaustive state coverage is needed.

The key test: **"Does this logic require zero I/O and zero collaborators?**" If yes, a Small (unit) test is appropriate. If any I/O is involved, use a Service Test.

### Benefits

- **High refactoring resilience.** Internal restructuring (renaming packages, merging classes, extracting functions) does not break tests because tests only touch the API surface.
- **Real confidence.** Tests exercise actual networking, actual SQL, actual serialization — catching the bugs that unit tests miss by design.
- **Reasonable speed.** With Testcontainers singleton patterns, Service Tests complete in seconds, not minutes.

---

## The Testing Trophy (Web Applications)

**The recommended shape for frontend-heavy applications.**

```
       ┌───────────┐
      │    E2E    │         ← Targeted critical journeys
     ┌┘           └┐
    │ Integration  │        ← Component + API testing
   ┌┘   Tests      └┐
  │                  │
  │  Static Analysis │      ← TypeScript, ESLint, type checking
  └──────────────────┘
```

### Principle

**Static analysis and user-integration testing** catch the most bugs per unit of effort in web applications. The type system prevents entire categories of runtime errors, and integration tests validate user-visible behavior through real component rendering.

### Layer Distribution

| Layer | Investment | What It Covers |
|-------|-----------|----------------|
| **Static Analysis** | High (automated, always-on) | Type mismatches, unused variables, import errors, accessibility violations, style linting |
| **Integration Tests** | High | Component rendering with real state, user interactions, API integration via MSW/Prism, route transitions |
| **E2E Tests** | Targeted | Critical user journeys: authentication, payment, data creation. Only paths where integration tests lack sufficient confidence. |
| **Unit Tests** | Low | Utility functions, formatters, parsers. Same rule as Honeycomb: no I/O, no collaborators. |

### Web-Specific Tooling

| Tool | Purpose |
|------|---------|
| **TypeScript strict mode** | Catches type errors at compile time. Eliminates entire bug categories. |
| **ESLint + accessibility rules** | Enforces code quality and WCAG compliance statically. |
| **React Testing Library** | Tests components from the user's perspective (queries by role, label, text — not implementation details). |
| **MSW (Mock Service Worker)** | Intercepts network requests at the service worker level for integration tests. |
| **Playwright / Cypress** | E2E testing against a full browser. Use for critical journeys only. |

---

## The Testing Pyramid (Legacy/Monoliths)

The traditional testing pyramid (many unit tests, fewer integration, fewest E2E) was designed for monolithic applications where:

- Internal logic was complex and deeply nested
- Integration boundaries were few (one database, one external API)
- Deployment was infrequent (monthly/quarterly)

### When the Pyramid Still Applies

- **Legacy monoliths** with deep, algorithmic business logic and few external integrations.
- **Embedded systems** where hardware constraints limit container-based testing.
- **Library development** where the public API surface is small and internal logic is complex.

### When the Pyramid Fails

For microservices and modern web applications, the pyramid over-invests in unit tests (which mock away the interesting behavior) and under-invests in integration tests (which catch the real bugs). Prefer the Honeycomb or Trophy instead.

---

## Shape Selection Decision Matrix

| Architecture | Recommended Shape | Primary Investment | Reasoning |
|-------------|------------------|-------------------|-----------|
| **Microservices (Go, Python backends)** | Honeycomb | Service Tests | Behavior lives at boundaries |
| **Web Applications (React, Next.js)** | Trophy | Static Analysis + Integration | Type system prevents most bugs |
| **Monolithic APIs** | Pyramid (modified) | Unit + Integration | Deep internal logic |
| **ML Pipelines** | Honeycomb variant | Service + Data validation | Model behavior at API boundary |
| **Event-Driven Systems** | Honeycomb + CDCT | Service Tests + Contracts | Async boundaries need contract validation |

---

## Common Shape Antipatterns

### The Ice Cream Cone

```
  ┌──────────────────────┐
  │   Massive E2E Suite  │    ← Slow, flaky, expensive
  └┐                    ┌┘
   │  Few Integration  │     ← Under-invested
   └┐                ┌┘
    │  Minimal Unit │         ← Also under-invested
    └──────────────┘
```

**Problem:** The suite takes hours to run, is riddled with flaky tests, and provides feedback too late to be useful. Developers start ignoring the suite and deploy without waiting.

**Fix:** Invert the cone. Move most E2E scenarios down to Service Tests using Testcontainers. Keep only critical user journeys as E2E.

### The Solitary Unit Fortress

```
    ┌─────┐
   │ E2E │
   └┐   ┌┘
    │   │ ← Almost nothing
    └┐ ┌┘
  ┌──────────────────────┐
  │ Thousands of Unit    │    ← Mocks everything
  │ Tests with Mocks     │    ← Breaks on every refactor
  └──────────────────────┘
```

**Problem:** Every internal class is mocked in isolation. Tests are extremely coupled to implementation details. Any refactor (renaming, extracting, merging) breaks dozens of tests — even when the external behavior is unchanged. Meanwhile, real integration bugs ship to production because no test exercises real boundaries.

**Fix:** Delete mocked unit tests that merely verify method call sequences. Replace with Service Tests that test the same behavior through the real API against real containers.

### The "We Only Have Unit Tests" Fallacy

**Problem:** A team reports "95% test coverage" but all tests are solitary unit tests with mocked dependencies. The coverage metric creates false confidence — the tests execute lines but validate nothing about real system behavior.

**Diagnosis:** Run mutation testing. If the mutation score is below 50% despite high coverage, the assertions are weak or testing the wrong things.

**Fix:** Measure mutation score alongside coverage. Replace mock-heavy unit tests with Service Tests. Use the Risk Matrix to prioritize which modules need the deepest testing.

---

## See Also

- `validation-layers.md` — Test types that compose each shape
- `frontend-testing.md` — Applying the Testing Trophy in React applications
- `worked-examples.md` — Shapes applied to concrete features
- `pipeline-architecture.md` — How shape choice affects pipeline composition
