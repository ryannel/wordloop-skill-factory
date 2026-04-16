# AI-Augmented Quality Engineering

This module defines how to leverage agentic AI for test maintenance, synthetic data generation, and automated test quarantine — transforming quality assurance from a manual bottleneck into an autonomous, self-healing discipline.

---

## Table of Contents

1. [Core Principle](#core-principle)
2. [Agentic Self-Healing Tests](#agentic-self-healing-tests)
3. [Synthetic Data Generation](#synthetic-data-generation)
4. [Automated Test Quarantine (Test SRE)](#automated-test-quarantine-test-sre)
5. [AI-Assisted Test Generation](#ai-assisted-test-generation)
6. [Guardrails and Limits](#guardrails-and-limits)

---

## Core Principle

AI agents handle the **maintenance and data requirements** of test suites, freeing engineers to focus on test architecture and risk assessment. The goal is not to replace human judgment in test design, but to automate the operational toil that makes test suites decay over time.

Three domains where AI augmentation provides the highest return:

1. **Self-Healing:** Automatically adapting tests when UI elements evolve, reducing maintenance by up to 80%.
2. **Synthetic Data:** Generating production-realistic datasets that are PII-free, edge-case-rich, and scalable.
3. **Quarantine Automation:** Autonomously isolating flaky tests from the critical pipeline path while preserving visibility.

---

## Agentic Self-Healing Tests

### The Problem

UI tests are fragile. When a button moves, a CSS class changes, or a DOM structure is refactored, selectors break — even if the user-facing behavior is identical. This creates a "test maintenance tax" that grows linearly with suite size.

### The Solution

AI agents reason through the DOM to recognise **contextual similarities** when UI elements evolve. Instead of failing because `#submit-btn` became `#primary-action`, the agent identifies the element by its visual context, label text, position, and role.

### Implementation Strategy

| Approach | Description | Trade-off |
|----------|-------------|-----------|
| **Semantic Selectors** | Use `data-testid`, `aria-label`, and role attributes as primary selectors. These are stable contracts between dev and test. | Requires developer discipline to maintain test IDs. |
| **Visual AI Matching** | AI compares screenshots to identify elements by visual appearance rather than DOM path. | Higher compute cost. Best for critical user journeys. |
| **Hybrid Approach** | Semantic selectors as primary, with AI fallback when selectors break. Agent proposes selector fixes for human review. | Best balance of speed and resilience. |

### Agent Workflow for Self-Healing

```
Test Fails → Selector Not Found
  │
  ├── Agent analyzes the current DOM
  ├── Agent searches for contextually similar elements
  │   (same label text, similar position, matching role)
  ├── Agent identifies candidate replacements
  │   ranked by confidence score
  │
  ├── Confidence > 90%?
  │   ├── Yes → Execute test with new selector, report success + proposed fix
  │   └── No  → Mark test as "needs-human-review", skip gracefully
  │
  └── Agent creates PR with selector updates for human approval
```

### Best Practices

- **Always use `data-testid` attributes** as the primary selector strategy. These are explicit contracts that survive refactoring.
- **Reserve AI healing for visual regression**, not logical assertion failures. If the expected behavior changed, the test should fail.
- **Log every healing action.** Self-healing is a diagnostic signal, not silent suppression. Track healing frequency per component to identify unstable UI areas.

---

## Synthetic Data Generation

### The Problem

Production databases contain rich, complex data that exercises real edge cases. But production data cannot be used in tests due to PII, compliance, and data volume constraints. Manually crafted fixtures are sparse, unrealistic, and miss the edge cases that cause production bugs.

### The Solution

Use AI to generate datasets that **mimic production complexity** while being fully synthetic — no PII, no compliance risk, infinitely scalable.

### Generation Strategies

| Strategy | Description | Best For |
|----------|-------------|----------|
| **Schema-Driven** | Generate data from database schemas, OpenAPI specs, or Pydantic models. AI fills fields with realistic values matching type constraints. | Service Test fixtures, API testing |
| **Distribution-Matching** | Analyze production data distributions (anonymized) and generate synthetic data with matching statistical properties. | Load testing, performance benchmarking |
| **Adversarial** | Generate edge cases designed to break validation logic: empty strings, unicode extremes, SQL injection attempts, max-length fields. | Security testing, fuzzing, boundary testing |
| **Scenario-Based** | Generate complete user journey datasets with referential integrity across tables (users → meetings → tasks → transcriptions). | System Test fixtures, E2E scenarios |

### Implementation Example

```python
from pydantic import BaseModel, Field
from typing import Literal

class SyntheticMeeting(BaseModel):
    """Schema for generating realistic meeting fixtures."""
    id: str = Field(pattern=r"^mtg_[a-z0-9]{8}$")
    title: str = Field(min_length=3, max_length=100)
    status: Literal["active", "completed", "cancelled"]
    duration_minutes: int = Field(ge=5, le=480)
    participant_count: int = Field(ge=1, le=50)
    created_by: str = Field(pattern=r"^usr_[a-z0-9]{8}$")

def generate_meeting_fixtures(count: int = 100) -> list[SyntheticMeeting]:
    """
    Generate realistic meeting fixtures.
    
    AI generates values that are:
    - Statistically distributed (most meetings 30-60 min, few are 5 or 480)
    - Contextually coherent (title matches typical meeting names)
    - Referentially valid (created_by references a valid user ID)
    """
    ...
```

### Privacy Controls

- **Never use production data directly.** All synthetic data must be generated, not copied and anonymized.
- **Validate PII absence.** Run PII detection tools on generated datasets before committing to test fixtures.
- **Maintain referential integrity.** Synthetic datasets must have valid foreign key relationships across tables.

---

## Automated Test Quarantine (Test SRE)

### The Problem

Flaky tests — tests that pass and fail non-deterministically — are the single largest threat to pipeline velocity. A single flaky test can block an entire team's deployment. But disabling flaky tests permanently hides real bugs.

### The Solution: Test SRE

Treat test health like service health. AI agents autonomously **demote flaky tests** from "blocking" to "quarantined" the moment they exhibit non-deterministic behavior. The owning team is automatically notified via the test's FQN ownership mapping.

### Quarantine Workflow

```
Test Exhibits Flakiness (passes then fails without code change)
  │
  ├── Agent calculates flakiness score
  │   (failures / total runs over rolling window)
  │
  ├── Flakiness Score > threshold (e.g., > 5%)?
  │   ├── Yes → Demote to quarantined
  │   │   ├── Remove from blocking pipeline
  │   │   ├── Move to non-blocking "quarantine" suite
  │   │   ├── Create ticket for owning team (via FQN lookup)
  │   │   └── Continue running in quarantine for monitoring
  │   │
  │   └── No → Keep in blocking pipeline, monitor
  │
  └── Quarantined test stabilizes (0% flakiness over N runs)?
      └── Agent proposes promotion back to blocking suite
```

### Flakiness Score Calculation

```
Flakiness Score = (inconsistent_results / total_runs) over rolling_window

Where:
  inconsistent_results = runs where the same commit produces both pass and fail
  total_runs = all runs in the rolling window (e.g., last 50 runs)
  threshold = 0.05 (5%)
```

### FQN-Based Ownership Routing

The Fully Qualified Name of a quarantined test maps directly to team ownership:

```
golang.service_test.infrastructure.database.user_repository.*
  → Team: core-backend
  → Slack: #core-eng-alerts
  → Ticket: JIRA project CORE

python.service_test.domain.transcription.*
  → Team: ml-platform
  → Slack: #ml-eng-alerts
  → Ticket: JIRA project ML
```

### Key Principle

**Pipeline velocity is fiercely protected while retaining visibility on technical debt.** Quarantine is not ignore — it is a managed degradation that prevents flaky tests from blocking the entire organization while ensuring the owning team is accountable for resolution.

---

## AI-Assisted Test Generation

### When to Use

- **Gap Detection:** AI analyzes code coverage reports and mutation testing results to identify untested paths and propose test cases.
- **Boilerplate Reduction:** AI generates the structural scaffolding for Testcontainers setup, fixture creation, and assertion boilerplate.
- **Edge Case Discovery:** AI reasons about input boundaries, type constraints, and failure modes to suggest test scenarios humans might miss.

### When NOT to Use

- **Test Architecture Decisions:** AI should not decide which validation layer, emulation strategy, or risk classification to apply. These are architectural decisions requiring human judgment.
- **Assertion Logic:** AI can suggest assertions, but the human must validate that assertions test meaningful behavior, not implementation details.
- **Contract Definitions:** Consumer expectations in CDCT must be defined by the humans who understand the consumer's actual requirements.

---

## Guardrails and Limits

| Guardrail | Rationale |
|-----------|-----------|
| **AI proposes, humans approve** | Self-healing selector changes, quarantine decisions, and generated tests all require human review before permanent adoption. |
| **Log every AI action** | Full audit trail of healing, quarantine, and generation decisions. No silent suppression of failures. |
| **Bound the blast radius** | AI agents can quarantine tests but cannot delete them. They can propose selector fixes but cannot merge them. |
| **Monitor heal rates** | A component with > 20% heal rate per sprint indicates a systemic stability problem that needs architectural attention, not more AI healing. |

---

## See Also

- `test-data-management.md` — Synthetic data integration with fixture factories
- `mutation-testing.md` — AI-assisted gap detection via mutation results
- `pipeline-architecture.md` — Quarantine integration with pipeline gates
