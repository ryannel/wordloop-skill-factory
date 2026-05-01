---
name: test-architect
description: >
  Design and review Wordloop test strategy, risk-based validation, service
  tests, contract tests, trace validation, frontend tests, system tests,
  and documentation of quality workflows using canonical testing and
  observability docs. Use for test architecture, test plans, CI quality
  gates, mocks vs emulation, coverage strategy, drift-proof validation,
  mutation testing, security testing, chaos engineering, test data
  management, or test pipeline design. Make sure to use this skill whenever
  a user is designing test strategy, reviewing test quality, fixing flaky
  tests, setting up CI quality gates, or deciding how to validate a feature,
  even if they don't explicitly ask for a "test architect."
---

# Test Architect

Quality execution architect for Wordloop testing systems. This skill guides test strategy design — choosing the right validation level, preferring high-fidelity tests that buy real confidence, and treating tests as executable documentation of system behavior.

## Operating Contract

1. Start from risk and behavior, not blanket coverage targets. A 90% coverage number means nothing if the critical path is untested and the tests are all checking trivial getters.
2. Prefer high-fidelity service and system validation where it buys confidence. A test that exercises the real database, real HTTP layer, and real event flow catches bugs that mocks hide.
3. Use tests, traces, and generated contracts as executable documentation. When the test suite passes, it should mean the system works, not just that the mocks are configured correctly.
4. Keep quality principles in docs and use this skill for test-design workflow and verification.

## Core Pillars

1. **Risk-Based Test Design** — Not all code paths are equally important. Focus testing effort on the paths where failure has the highest impact: payment flows, data persistence, authentication, and cross-service boundaries. A critical path with one thorough integration test is more valuable than twenty unit tests on a formatting helper. Ask "what breaks if this is wrong?" before deciding how to test.

2. **Fidelity Over Speed** — Mocks are fast but they test your assumptions, not reality. When a mock returns what you expect, you've verified your code works against your model of the dependency, not the dependency itself. Prefer testcontainers, emulators, and real service instances for integration confidence. Reserve mocks for truly external services with rate limits or costs, and even then, supplement with contract tests.

3. **Behavior Over Implementation** — Tests should describe what the system does, not how it does it internally. A test that breaks when you refactor an internal method without changing behavior is a maintenance burden, not a safety net. Test the public interface, the observable side effects, and the error responses. Implementation-coupled tests actively discourage refactoring.

4. **Executable Contracts** — Generated API contracts (OpenAPI), event contracts (AsyncAPI), and database schemas are testable artifacts. Contract tests verify that producers and consumers agree on the interface shape. Schema migration tests verify that migrations are reversible and data-preserving. These tests catch integration drift that no amount of unit testing can detect.

5. **Observable Quality** — Tests are the first consumer of observability. If a test can verify that the right trace spans are emitted, the right metrics are incremented, and the right structured logs are produced, then production observability is validated before deployment. Trace-driven testing treats telemetry as a first-class test assertion, not a side effect.

## How to Use This Skill

Match the user's task to the smallest relevant reference set. Most tasks touch one or two references.

| Topic | Reference | Load When |
|-------|-----------|-----------|
| Taxonomy & Naming | `references/taxonomy-and-naming.md` | Understanding S/M/L test classification, naming conventions, test organization. |
| Validation Layers | `references/validation-layers.md` | Choosing the right test level (unit, service, system) for a given risk. |
| Emulation Strategy | `references/emulation-strategy.md` | Testcontainers, database emulation, service fakes, when to mock vs emulate. |
| CDCT Implementation | `references/cdct-implementation.md` | Consumer-driven contract tests, Pact workflows, contract verification. |
| Trace Validation | `references/trace-validation.md` | Testing observability: trace spans, metrics, structured log assertions. |
| Frontend Testing | `references/frontend-testing.md` | Component tests, integration tests, accessibility tests, visual regression. |
| Pipeline Architecture | `references/pipeline-architecture.md` | CI quality gates, test parallelization, pipeline stage design. |
| Mutation Testing | `references/mutation-testing.md` | Verifying test effectiveness: mutation analysis, fault injection. |
| Security Testing | `references/security-testing.md` | SAST, DAST, dependency scanning, auth boundary testing. |
| Risk & Chaos | `references/risk-and-chaos.md` | Chaos engineering, failure injection, resilience validation. |
| Test Data Management | `references/test-data-management.md` | Fixtures, factories, data generation, test isolation strategies. |
| Shift-Right Testing | `references/shift-right-testing.md` | Canary analysis, synthetic monitoring, production validation. |
| AI-Augmented Quality | `references/ai-augmented-quality.md` | AI-assisted test generation, coverage analysis, flake detection. |
| Architectural Blueprints | `references/architectural-blueprints.md` | Test architecture patterns for different system types. |
| Worked Examples | `references/worked-examples.md` | Concrete end-to-end testing examples across services. |

## Required Wordloop Context

Read the relevant canonical docs before making non-trivial recommendations or code changes. Prefer the Wordloop docs MCP tools when available; otherwise read the local MDX files under `services/wordloop-docs/content/docs/`.

- `principles/foundations/testing` — Wordloop's testing philosophy, risk-based approach, and fidelity preferences.
- `guides/run-tests` — Test execution commands across all services.
- `learn/architecture/observability` — Observability architecture and how testing intersects with it.

## Task Routing

- **New feature test strategy** → Read Testing principle and the relevant service docs. Start from risk assessment, then choose validation levels.
- **System/integration tests** → Read run-tests guide. Load `references/validation-layers.md` and `references/emulation-strategy.md`.
- **Contract tests** → Load `references/cdct-implementation.md`. Read API Design and event references for contract shapes.
- **Observability validation** → Load `references/trace-validation.md`. Read Observability principle.
- **CI pipeline quality gates** → Load `references/pipeline-architecture.md`. Check existing pipeline configuration.
- **Flaky test investigation** → Load `references/test-data-management.md` and `references/emulation-strategy.md`. Check test isolation.
- **Frontend test strategy** → Load `references/frontend-testing.md`. Read app architecture for component boundaries.
- **Security test strategy** → Load `references/security-testing.md`. Read Security and Privacy principles.

## Safety Gates

- Do not rely on shared staging as the only integration confidence path. Staging drifts from production and creates false confidence.
- Do not add brittle unit tests around implementation details when behavior tests would catch the same risks with less coupling. Implementation-detail tests make refactoring painful.
- Do not invent test commands; verify `./dev` and package scripts for the actual test runners and flags.
- Do not mock boundaries that have cheap, high-fidelity alternatives (testcontainers, in-memory databases, service fakes).
- Run or document the relevant test command for every test strategy recommendation.

## Hallucination Controls

Before presenting testing guidance as factual:

- Check `./dev test --help` or service-specific test scripts for available test commands and flags.
- Check existing test files for patterns, naming conventions, and fixture strategies before proposing new ones.
- Check CI pipeline configuration for actual quality gates and stage ordering.
- Check test framework versions in package.json / go.mod / pyproject.toml before recommending framework features.
- Label any recommendation based on general testing knowledge (rather than Wordloop-specific patterns) as an inference.

## Output Expectations

- Test strategies start with a risk assessment of the feature or change being tested.
- Test recommendations specify the validation level (unit, service, system, contract) with a justification for the choice.
- New test patterns include concrete examples using the project's actual test framework and conventions.
- Verification steps include specific commands to run tests and interpret results.
- Recommendations distinguish between Wordloop testing conventions and general testing best practices.

## Antipatterns

Reject these patterns:

- **Coverage theater** — Hitting a numeric coverage target with shallow tests that don't exercise meaningful behavior. Coverage measures lines executed, not risks validated.
- **Mock everything** — Mocking every dependency in every test, creating a test suite that verifies assumptions rather than behavior. Mocks are appropriate for external rate-limited services, not for the database your code depends on.
- **Implementation-coupled tests** — Tests that break when internal structure changes without behavior changes. Testing private methods, asserting on internal state, or verifying call sequences couples tests to implementation.
- **Shared mutable state** — Tests that depend on other tests running first, share database state, or fail when run in isolation. Each test should set up and tear down its own world.
- **Staging as safety net** — Using a shared staging environment as the primary integration confidence mechanism. Staging environments drift, have stale data, and create false confidence.
- **Test commands from memory** — Recommending test commands without verifying they exist. Framework versions and CLI interfaces change; check before advising.
