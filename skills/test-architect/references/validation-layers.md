# The Validation Grid

This module defines the four test types organized across two axes: **Scope** (Service vs System) and **Fidelity** (Emulated vs Live). Each type has distinct execution constraints, cost profiles, and validation targets. The types are additive — higher-fidelity types assume lower-fidelity types pass.

---

## Table of Contents

1. [The 2×2 Grid](#the-2x2-grid)
2. [Automated Tests (Emulated)](#automated-tests-emulated)
   - [Service Test](#service-test)
   - [System Test](#system-test)
3. [Manual Tests (Live)](#manual-tests-live)
   - [Live Service Test](#live-service-test)
   - [Live System Test](#live-system-test)
4. [Comparison Matrix](#comparison-matrix)
5. [Decision Flowchart](#decision-flowchart)

---

## The 2×2 Grid

Tests are classified along two independent axes:

- **Scope:** How many services are involved? (Service = one, System = all)
- **Fidelity:** Are 3rd-party dependencies emulated or live?

```
                     │  Emulated (Automated)     │  Live (Manual Only)
─────────────────────┼───────────────────────────┼──────────────────────────
 Service (single)    │  Service Test             │  Live Service Test
                     │  Every PR. Hermetic.      │  Dev iteration. Cost-bearing.
─────────────────────┼───────────────────────────┼──────────────────────────
 System  (multi)     │  System Test              │  Live System Test
                     │  Merge to trunk. Traces.  │  Holistic check. Cost-bearing.
```

### The Two Execution Models

**Automated (left column):** These tests run in CI/CD pipelines. They are hermetic — no external network calls, no cloud credentials, no cost. They form the backbone of continuous validation.

**Manual (right column):** These tests call real 3rd-party APIs. They are never automated. A developer runs them locally or via manual dispatch when actively working on an integration or validating before a release.

---

## Automated Tests (Emulated)

### Service Test

The **"Sociable" Foundation**. This is the primary validation type and the minimal unit of meaningful testing for microservices. Service Tests validate the microservice from its API entry point through business logic to real, ephemeral persistence.

#### Scope

- **Own Service:** Fully real. Running from its HTTP/gRPC entry point.
- **Own Dependencies:** Real, ephemeral instances via **Testcontainers** (PostgreSQL, Redis, Kafka, etc.).
- **3rd-Party APIs:** High-fidelity emulator containers (**Prism** for OpenAPI mocking, **LocalStack** for AWS, **Ollama** for LLM APIs).

#### Why "Sociable"

Unlike "solitary" unit tests that mock every collaborator, Service Tests exercise the real collaboration between internal components. The service's internal classes, repositories, and handlers all work together as they would in production. Only **external boundaries** are emulated.

#### Key Properties

- **Hermetic:** No external network calls. No cloud credentials required. Runs identically on developer laptops and CI runners.
- **Deterministic:** Container state is reset per suite. No test pollution.
- **Fast Feedback:** Targets < 5 seconds per test. Runs on every PR.
- **Refactor-Resilient:** Internal refactoring (renaming classes, restructuring packages) does not break tests as long as the API contract and outcomes remain intact.

#### What Service Tests Validate

- Request parsing and input validation
- Business rule enforcement
- Database schema correctness (real SQL against real PostgreSQL)
- Serialization/deserialization fidelity
- Error response formatting
- Event emission (WebSocket, Pub/Sub) with correct payloads
- Authorization and authentication logic (against emulated identity providers)

#### Example Stack (Go Microservice)

```
┌─────────────────────────────────┐
│ Test Runner (go test)           │
│  → HTTP Client sends requests   │
│  → Asserts responses + DB state │
├─────────────────────────────────┤
│ Service Under Test              │
│  → Real HTTP handlers           │
│  → Real business logic          │
│  → Real repository layer        │
├─────────────────────────────────┤
│ Testcontainers                  │
│  ├── PostgreSQL (ephemeral)     │
│  ├── Redis (ephemeral)          │
│  ├── Prism (OpenAPI mock)       │
│  └── LocalStack (AWS services)  │
└─────────────────────────────────┘
```

---

### System Test

Wires **multiple internal microservices** and their hosted dependencies together in an ephemeral environment. All external 3rd-party services remain emulated.

#### Scope

- **Internal Services:** All services are live in an ephemeral Docker/Kubernetes environment.
- **Own Dependencies:** Real databases, caches, message brokers — all via Testcontainers or Docker Compose.
- **3rd-Party APIs:** All directed to high-fidelity emulator containers.

#### Core Requirement: Trace-Driven Validation

**System Tests must validate that OpenTelemetry traces are unbroken end-to-end.** This means:

- Every service correctly propagates `traceparent` context.
- Span parent-child relationships are correct across service boundaries.
- No spans are missing from the expected trace tree.
- TraceIDs are not lost between services.

A System Test that does not validate trace completeness is incomplete.

#### What System Tests Validate

- Cross-service communication protocols (HTTP, gRPC, WebSocket, Pub/Sub)
- Event propagation chains (Service A emits → Service B consumes → Service C reacts)
- Distributed transaction consistency
- End-to-end user journeys through multiple services
- Infrastructure resilience (via chaos experiments — see `risk-and-chaos.md`)

#### Example Stack (Multi-Service Platform)

```
┌──────────────────────────────────────────────────────┐
│ Test Runner (pytest / go test / Playwright)          │
│  → Triggers user journeys                            │
│  → Asserts outcomes across services                  │
│  → Validates OTel traces end-to-end                  │
├──────────────────────────────────────────────────────┤
│ Service Cluster (Ephemeral)                          │
│  ├── wordloop-core (Go)                              │
│  ├── wordloop-ml (Python/FastAPI)                    │
│  └── wordloop-app (Next.js)                          │
├──────────────────────────────────────────────────────┤
│ Infrastructure (Testcontainers / Docker Compose)     │
│  ├── PostgreSQL                                      │
│  ├── Redis                                           │
│  ├── OTel Collector (Jaeger / in-memory exporter)    │
│  ├── LocalStack (S3, SQS)                            │
│  └── Prism / WireMock (3rd-party API emulation)      │
└──────────────────────────────────────────────────────┘
```

---

## Manual Tests (Live)

### The Shared Philosophy

The Live Service Test and Live System Test exist to solve the **same fundamental problem**: flushing out issues that only surface when integrating with a real 3rd-party system. Emulators like Prism, LocalStack, and Ollama are excellent for hermetic testing, but they cannot replicate every nuance of a live provider — real rate limiting, real credential flows, real model behavior, real state mutations.

The two live types differ only in **blast radius**, not in purpose:

| | Live Service Test | Live System Test |
|---|---|---|
| **Scope** | Single service + live 3rd parties | Full service cluster + live 3rd parties |
| **Iteration Speed** | Fast — used heavily while actively developing an integration | Slower — used to validate the whole before a release |
| **Typical Use** | "I'm working on the OpenAI integration and need to run this repeatedly as I iterate" | "The integration is ready — let me confirm everything works end-to-end" |

#### Shared Constraints

Both live types share identical execution constraints because they share the same root concerns — cost accrual from real API calls and the complexity of driving state in external systems:

- **Always Manual.** Live tests are never automated in CI/CD pipelines. They are triggered exclusively by a developer running them locally or via manual dispatch. Automating live tests risks uncontrolled cost and unintended state mutations in 3rd-party systems.
- **Cost-Bearing.** Every execution incurs real API charges. Developers must budget accordingly and be intentional about when and how often they run these tests.
- **Stateful Side Effects.** Real 3rd-party calls can create real state — OAuth tokens, payment charges, model inference logs. Tests must be designed to be safely re-runnable: use unique identifiers, clean up after execution, and avoid operations that cannot be undone.
- **No Production Databases.** Despite using live 3rd-party APIs, live tests never connect to production data stores.

---

### Live Service Test

A Service Test where emulators are selectively replaced with **real, external 3rd-party endpoints** for the specific integration being developed or validated.

#### Scope

- **Own Service:** Fully real, same as Service Test.
- **Own Dependencies:** Testcontainers, same as Service Test.
- **3rd-Party APIs:** Selected real endpoints (e.g., OpenAI API, Google OAuth, Stripe).

#### When to Use

The Live Service Test is the developer's primary tool for **iterating on a live integration**. Use it when:

- **Developing a new integration:** You are building or modifying the code that talks to a 3rd-party API and need to run against the real provider repeatedly as you work.
- **LLM prompt engineering:** Prompt quality evaluation requires real model behavior. Local models (Ollama) approximate but don't replicate production LLM responses for nuanced evaluation.
- **Closed SDKs:** The 3rd-party provides a proprietary SDK with no official test container or emulator (e.g., certain payment processors, identity providers).
- **Credential verification:** Validating that production API keys, OAuth credentials, and rate limit handling work against the real provider.

#### Example: LLM Integration Validation

```python
# Live Service Test — validates real LLM behavior
# Trigger: manual — run locally during integration development
# Cost: ~$0.05 per execution

def test_transcription_summary_should_capture_key_topics_when_meeting_audio_provided():
    """
    Uses real OpenAI API to validate prompt engineering produces
    meaningful summaries. Ollama approximation is insufficient for
    evaluating nuanced topic extraction quality.
    """
    response = service.summarize(audio_fixture="team_standup_5min.wav")
    assert "action items" in response.summary.lower()
    assert response.confidence_score > 0.75
```

---

### Live System Test

A System Test where high-confidence emulators are replaced with selected live 3rd-party endpoints. This is the broadest-scoped validation — confirming that the **entire integrated system** works against real providers.

#### Scope

- **Internal Services:** All live, same as System Test.
- **Own Dependencies:** Real databases and infrastructure, same as System Test.
- **3rd-Party APIs:** Selected real endpoints for critical integrations, with emulators retained where the real service adds no additional confidence (e.g., S3 via LocalStack remains sufficient).

#### When to Use

The Live System Test is used when the integration work is **complete and you need holistic confidence**:

- The integration code has been validated via the Live Service Test — now confirm it works within the full service cluster.
- Pre-release validation for major deployments.
- Regulatory compliance testing requiring proof of real end-to-end integration.
- Verifying production provider behavior (rate limiting, geographic routing) in a multi-service context.

---

## Comparison Matrix

| Property | Service Test | System Test | Live Service Test | Live System Test |
|----------|-------------|------------|-------------------|-----------------|
| **Purpose** | Service validation | Cross-service validation | Iterate on live integration | Holistic live validation |
| Services Under Test | 1 | All | 1 | All |
| Own Dependencies | Testcontainers | Testcontainers / Compose | Testcontainers | Testcontainers / Compose |
| 3rd-Party APIs | Emulated | Emulated | **Selected Live** | **Selected Live** |
| Cost | Free | Free | Per-call billing | Per-call billing |
| Execution | **Automated** (every PR) | **Automated** (merge to trunk) | **Manual only** | **Manual only** |
| Trace Validation | Optional | **Mandatory** | Optional | **Mandatory** |
| Hermetic | Yes | Yes | No | No |
| Target Duration | < 5s/test | < 5min/suite | < 30s/test | < 15min/suite |

---

## Decision Flowchart

Use this decision tree when choosing which test type to apply:

```
START: What are you testing?
  │
  ├─ Pure algorithmic logic, no I/O?
  │   └─ Small (Unit) Test — no type classification needed
  │
  ├─ Single service, own logic + dependencies?
  │   └─ Service Test (automated, every PR)
  │
  ├─ Multiple services communicating?
  │   └─ System Test (automated, merge to trunk)
  │       └─ Include OTel trace assertions (always mandatory)
  │
  └─ Need to validate against a real 3rd-party API?
      │
      ├─ Actively iterating on the integration code?
      │   └─ Live Service Test (manual, run locally)
      │
      └─ Integration is ready, need holistic confidence?
          └─ Live System Test (manual, run locally)
```

---

## See Also

- `emulation-strategy.md` — How to configure emulator containers for Service and System Tests
- `test-data-management.md` — Database reset and fixture strategies per test type
- `worked-examples.md` — End-to-end walkthroughs applying the Validation Grid
- `pipeline-architecture.md` — How each test type maps to CI/CD stages
