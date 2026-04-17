# Debt-Aware Architecture Decision Records

Architecture Decision Records (ADRs) are the primary vehicle for capturing the *why* behind a system's design. Every significant architectural choice — technology selection, service boundary definition, data model design — is recorded as an immutable entry in the decision log. The Debt-Aware model extends the standard ADR format by treating every decision as a financial transaction with explicit costs, risks, and remediation plans.

---

## Table of Contents
1. [Philosophy](#philosophy)
2. [Append-Only Governance](#append-only-governance)
3. [The Debt-Aware Model](#the-debt-aware-model)
4. [Status Lifecycle](#status-lifecycle)
5. [ADR Template](#adr-template)
6. [Writing Guidance](#writing-guidance)
7. [Debt Categorization](#debt-categorization)
8. [Review Cadence](#review-cadence)
9. [ADR Log Organization](#adr-log-organization)
10. [Worked Example](#worked-example)

---

## Philosophy

ADRs answer the question every engineer eventually asks: "Why is it done this way?" Without an ADR, the answer is lost in Slack threads, meeting notes, or the memory of people who may no longer be at the company. With an ADR, the answer is version-controlled, searchable, and immutable.

An ADR is not a design document. A design document proposes and explores options before a decision is made. An ADR records the final decision and its rationale *after* the decision is made. The two are complementary: the design document is the working draft, the ADR is the published record.

---

## Append-Only Governance

Once an ADR's status is set to **Accepted**, it is never modified. If a decision changes — because requirements evolved, better options emerged, or the original choice proved costly — a new ADR is created that **Supersedes** the old one.

### Why Append-Only?

Modifying historical ADRs destroys institutional knowledge. Six months later, no one can reconstruct why the team chose PostgreSQL over DynamoDB, because the original reasoning was overwritten when the team migrated. The append-only model preserves the full decision trail:

```
ADR-001: Use PostgreSQL for meeting data (Accepted)
    └── ADR-007: Migrate meeting data to Firestore (Accepted, Supersedes ADR-001)
        └── ADR-015: Add read replica for analytics queries (Accepted, Extends ADR-007)
```

Each ADR in the chain is a snapshot of the team's understanding at that point in time. Reading the chain reconstructs the full evolution of the architecture.

### Permitted Modifications to Accepted ADRs

Only two changes are permitted to an accepted ADR:

1. **Adding a "Superseded by" link** when a new ADR replaces this decision.
2. **Correcting factual errors** (typos, broken links) that do not change the decision or rationale. These corrections are treated as patches, not content changes.

---

## The Debt-Aware Model

Every architectural decision introduces trade-offs. The Debt-Aware model makes these trade-offs explicit by framing them as financial transactions:

| Component | Meaning | Example |
|-----------|---------|---------|
| **Decision (Principal)** | The choice made | "Use SQLite for local testing instead of a full PostgreSQL instance" |
| **Drawbacks (Interest)** | The ongoing costs of this choice | "SQLite does not support concurrent write access; tests must run serially" |
| **Risk (Multiplier)** | The probability and impact of the drawback failing | "High probability (team growing), moderate impact (CI pipeline slows to 10+ minutes)" |
| **Total Cost** | The full cost of the decision over time | `Principal + (Drawbacks × Risk)` |

### The Cost Formula

$$Total\ Cost = Principal + (Drawbacks \times Risk)$$

This formula is not a precision tool — it is a thinking framework. The purpose is to force explicit reasoning about trade-offs so that decisions are made with open eyes rather than optimistic assumptions.

### Applying the Model

When writing the Debt-Aware section of an ADR:

1. **Name the principal:** State the decision clearly and simply.
2. **Enumerate the drawbacks:** List every ongoing cost the decision introduces. Be honest — an ADR that lists no drawbacks for a significant decision is probably incomplete.
3. **Score the risk:** For each drawback, estimate the probability (Low / Medium / High) and the impact (Low / Medium / High) of the drawback becoming a real problem.
4. **Define the remediation:** Describe the trigger or plan for addressing each drawback. If there is no remediation plan, explicitly state why (e.g., "Accepted as permanent trade-off: the cost is justified by the benefit").

---

## Status Lifecycle

```
┌──────────┐     ┌──────────┐     ┌──────────────┐
│ Proposed  │────▶│ Accepted │────▶│ Superseded   │
└──────────┘     └──────────┘     │ (by ADR-XXX) │
      │                           └──────────────┘
      │
      ▼
┌──────────┐
│ Rejected │
└──────────┘
```

| Status | Meaning |
|--------|---------|
| **Proposed** | The ADR has been drafted and is under review. The decision has not been finalized. |
| **Accepted** | The decision has been reviewed, approved, and is now the architectural record of truth. |
| **Rejected** | The proposed decision was not approved. The ADR is preserved to document *why* the option was not chosen. |
| **Superseded** | A newer ADR has replaced this decision. The superseding ADR's number is linked. |
| **Deprecated** | The decision is no longer relevant because the system component it governs has been retired. |

---

## ADR Template

```markdown
# ADR-[NNN]: [Descriptive Title]

## Metadata
| Field | Value |
|-------|-------|
| **Status** | [Proposed / Accepted / Rejected / Superseded / Deprecated] |
| **Date** | [YYYY-MM-DD] |
| **Deciders** | [Names or roles of the decision-makers] |
| **Supersedes** | [ADR-XXX (if applicable)] |
| **Superseded by** | [ADR-XXX (if applicable)] |
| **Review Date** | [YYYY-MM-DD: date to re-evaluate this decision] |

## Context

[Describe the situation that requires a decision. What problem are we solving?
What constraints exist? What forces are in tension?]

## Decision Drivers

- [Driver 1: e.g., "Time-to-market pressure for the MVP"]
- [Driver 2: e.g., "Team has deep PostgreSQL expertise"]
- [Driver 3: e.g., "Must support global read replication"]

## Considered Options

### Option A: [Name]
[Brief description, pros, and cons]

### Option B: [Name]
[Brief description, pros, and cons]

### Option C: [Name]
[Brief description, pros, and cons]

## Decision

[State the decision clearly. Use active voice: "We will use X for Y."]

## Technical Debt Assessment

### Incurred Debt (Interest)
- [Drawback 1: ongoing cost introduced by this decision]
- [Drawback 2: ongoing cost]

### Risk (Multiplier)
| Drawback | Probability | Impact | Score |
|----------|-------------|--------|-------|
| [Drawback 1] | [Low/Med/High] | [Low/Med/High] | [L×L to H×H] |
| [Drawback 2] | [Low/Med/High] | [Low/Med/High] | [L×L to H×H] |

### Remediation Plan
- [When/how will each drawback be addressed?]
- [What trigger causes remediation to begin?]
- [If no remediation is planned, explain why]

### Total Cost Assessment
`Total Cost = [Decision] + ([Drawback costs] × [Risk])`
[One-sentence summary of the cost assessment]

## Consequences

### Positive
- [Benefit 1]
- [Benefit 2]

### Negative
- [Cost 1]
- [Cost 2]

## References
- [Link to design document]
- [Link to related ADR]
- [Link to relevant diagram]
```

---

## Writing Guidance

### Titles

ADR titles use the format `ADR-NNN: [Verb] [Object] [Context]`. The title should be specific enough that someone scanning the ADR log can understand the decision without opening the document.

**Good titles:**
- `ADR-001: Use Firestore for meeting state storage`
- `ADR-005: Adopt OAuth 2.1 with PKCE for public-facing authentication`
- `ADR-012: Split monolith meeting service into recording and transcription microservices`

**Bad titles:**
- `ADR-001: Database` (too vague)
- `ADR-005: Authentication update` (does not state the decision)
- `ADR-012: Architecture changes` (states nothing)

### Context Section

The context section sets the stage. It describes the world *before* the decision is made. Write it so that a new team member who was not present for the discussion can understand the constraints and tensions that led to this decision.

- Include quantitative data where available: "The current API serves 2,000 req/s with P99 latency of 450ms."
- Include organizational constraints: "The team has 3 Go engineers and no Python experience."
- Include temporal constraints: "The feature must ship before Q3 to meet the partnership commitment."

### Decision Section

State the decision in a single, unambiguous sentence. Use active voice:

- ✅ "We will use Firestore for meeting state storage."
- ❌ "It was decided that Firestore would be used" (passive, vague actor).
- ❌ "We should probably use Firestore" (hedging — an ADR records a decision, not a suggestion).

### Technical Debt Section

This is the distinguishing feature of a Debt-Aware ADR. Be brutally honest about the drawbacks. An ADR that lists only benefits is not an ADR — it is a sales pitch.

Every drawback must include:
- **What** the ongoing cost is (concrete, specific).
- **When** it is likely to manifest (under what conditions).
- **How severe** the impact is (Low / Medium / High).
- **What** the remediation plan is (or why none exists).

---

## Debt Categorization

When documenting technical debt, classify it by type to enable portfolio-level analysis:

| Debt Type | Meaning | Example |
|-----------|---------|---------|
| **Code Debt** | Shortcuts in implementation that increase maintenance cost | Missing test coverage, duplicated logic, fragile dependencies |
| **Infrastructure Debt** | Manual operations or non-standard deployments | Manual patching, snowflake server configurations |
| **Solution Debt** | Architectural choices that limit future system evolution | Tight coupling between services, monolithic data model |
| **Documentation Debt** | Missing or outdated documentation that increases onboarding time | Undocumented API endpoints, stale architecture diagrams |

Tagging ADRs with debt types enables teams to track debt distribution across categories and identify systemic patterns (e.g., "70% of our debt is infrastructure-related, suggesting we need to invest in platform automation").

---

## Review Cadence

Every accepted ADR includes a **Review Date** — a future date on which the team re-evaluates whether the decision and its associated debt are still appropriate.

### Setting the Review Date

| Decision Scope | Review Frequency | Rationale |
|----------------|-----------------|-----------|
| **Tactical** (library choice, build tool) | 6 months | Technology moves fast; alternatives may emerge |
| **Structural** (service boundaries, data model) | 12 months | Structural changes are expensive; annual review is sufficient |
| **Strategic** (platform choice, vendor commitment) | 18-24 months | Strategic commitments require contract-aligned review cycles |

### The Review Process

1. **Read the original ADR** and the debt assessment.
2. **Evaluate the drawbacks:** Have any of the predicted risks materialized? Has the interest rate changed?
3. **Assess alternatives:** Have better options emerged since the decision was made?
4. **Decide: Reaffirm, Extend, or Supersede.**
   - **Reaffirm:** The decision is still correct. Update the review date.
   - **Extend:** The decision is correct but the debt has grown. Create a remediation plan.
   - **Supersede:** The decision is no longer appropriate. Create a new ADR that supersedes this one.

---

## ADR Log Organization

### File Structure

```
docs/architecture/adrs/
├── index.md              # Chronological log with status badges
├── 001-use-firestore.md
├── 002-adopt-oauth21.md
├── 003-use-mermaid-diagrams.md
├── ...
└── NNN-latest-decision.md
```

### ADR Index

The `index.md` file provides a scannable log of all ADRs:

```markdown
# Architecture Decision Log

| # | Title | Status | Date | Supersedes |
|---|-------|--------|------|------------|
| 001 | [Use Firestore for meeting state](001-use-firestore.md) | ✅ Accepted | 2025-03-15 | — |
| 002 | [Adopt OAuth 2.1 with PKCE](002-adopt-oauth21.md) | ✅ Accepted | 2025-04-02 | — |
| 003 | [Use Mermaid.js for inline diagrams](003-use-mermaid-diagrams.md) | ✅ Accepted | 2025-05-10 | — |
| 004 | [Replace REST with gRPC for internal services](004-internal-grpc.md) | ❌ Rejected | 2025-06-01 | — |
| 005 | [Migrate meeting state to Spanner](005-migrate-to-spanner.md) | ✅ Accepted | 2025-08-20 | ADR-001 |
```

---

## Worked Example

### ADR-001: Use Firestore for Meeting State Storage

#### Metadata
| Field | Value |
|-------|-------|
| **Status** | Superseded |
| **Date** | 2025-03-15 |
| **Deciders** | Platform Team Lead, Backend Architect |
| **Superseded by** | ADR-005 |
| **Review Date** | 2025-09-15 |

#### Context

The meeting service needs a primary data store for meeting state (metadata, participants, schedule). The store must support:
- Sub-10ms read latency at P99
- Real-time change events for the WebSocket layer
- Global availability (multi-region)

The team has 4 engineers, all with prior Firestore experience. The launch deadline is 8 weeks away.

#### Decision Drivers
- Sub-10ms read latency requirement
- Real-time listener support (native to Firestore)
- Team expertise in Firestore
- 8-week launch deadline

#### Considered Options

**Option A: Firestore** — Managed NoSQL with native real-time listeners. Team has deep expertise. Scales automatically.

**Option B: PostgreSQL (Cloud SQL)** — Team has moderate SQL experience. Requires separate change-data-capture for real-time events. More operational overhead.

**Option C: Spanner** — Global consistency and relational model. Team has no experience. Higher cost and complexity for the current scale.

#### Decision

We will use Firestore for meeting state storage.

#### Technical Debt Assessment

**Incurred Debt:**
- Firestore's NoSQL model limits complex query patterns (no JOINs, limited aggregation).
- Real-time listeners at scale (>10,000 concurrent listeners) may hit Firestore's connection limits.

**Risk:**

| Drawback | Probability | Impact | Score |
|----------|-------------|--------|-------|
| Query limitations force denormalization | Medium | Medium | M×M |
| Concurrent listener limits hit | Low | High | L×H |

**Remediation Plan:**
- Query limitations: Acceptable for current data model. If analytics queries are needed, stream data to BigQuery via Cloud Functions. Trigger: when the first analytics feature is prioritized.
- Listener limits: Monitor concurrent listener count. If it exceeds 5,000, evaluate migration to Spanner (see Option C). Trigger: monitoring alert at 5,000 concurrent listeners.

**Total Cost:** `Firestore + (Query limitations × Medium) + (Listener limits × Low-High)`
The cost is acceptable given the team's expertise and the launch deadline. The debt is explicitly bounded by the remediation triggers.

#### Consequences

**Positive:**
- Launch within 8 weeks (team expertise eliminates learning curve)
- Native real-time support (no CDC pipeline needed)
- Zero operational overhead (fully managed)

**Negative:**
- Complex queries require denormalization or BigQuery offloading
- Vendor lock-in to Google Cloud Firestore data model

#### References
- [System Container Diagram](../diagrams/containers.d2)
- [Firestore Data Model](../data-model.md)
