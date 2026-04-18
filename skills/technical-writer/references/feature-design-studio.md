# Feature Design Studio Documentation

The Feature Design Studio is the canonical workspace for managing a feature through its full lifecycle: Specify → Discover → Design → Implement → Validate. Each phase has a structured document with clear conventions. This reference governs how to author and maintain those documents.

---

## Document Conventions

### State What the Design Is — Never What It Is Not

Design documents state decisions positively. They do not use comparative framing ("instead of X", "unlike Y", "rather than Z") in the body of the document.

**❌ Comparative framing:**
> Instead of proxying audio through Core, the client requests a short-lived signed URL and streams directly from Cloud Storage.

**✅ Positive statement:**
> Core generates a short-lived signed URL. The client streams audio directly from Cloud Storage using that URL, with standard HTTP range requests for seeking.

Rationale for rejected paths belongs in **Rejected Alternatives** — not in the prose describing the chosen design. A reader of the design document wants to understand what the system does. The Rejected Alternatives section is where they go to understand why other options were not chosen.

### Open Questions Are Either Resolved or Removed

The Open Questions section contains only unresolved decisions. When a question is answered, incorporate the decision into the relevant section of the document and **delete the question from the table**. Do not mark questions as "Resolved" and keep them — a resolved question is no longer open.

Open questions use a simple two-column table:

```markdown
| # | Question |
|---|----------|
| 1 | Should microphone and system audio capture be in the same phase? |
```

When all questions are resolved, replace the table with:

```markdown
_No open questions — all decisions have been resolved and incorporated into the spec above._
```

### Rejected Alternatives Are Not Prose

A rejected alternative is a brief entry in a table. It states what was considered and why it was not chosen. It does not need a heading, paragraphs, or diagrams.

```markdown
| Alternative | Why Rejected |
|------------|-------------|
| Core proxies audio playback from GCS | Unnecessary bandwidth consumption in Core; signed URLs give the client direct access to Cloud Storage |
```

---

## Specification Document (`specification.mdx`)

The specification answers: **What are we building, why, and what does success look like?**

### Required Sections

| Section | Purpose |
|---------|---------|
| **Problem Statement** | The user problem being solved and why it matters now |
| **Success Criteria** | Measurable outcomes — checkbox list |
| **Scope** | In Scope and Out of Scope — explicit, not implied |
| **User Stories** | Key user flows with Given/When/Then acceptance criteria |
| **Open Questions** | Unresolved decisions only — removed when answered |

### Writing Rules

- Success criteria are **measurable and binary** — they can be checked off. Avoid vague criteria like "improve performance" or "better UX."
- Scope is explicit on both sides. If it is not listed as In Scope, it is ambiguous. Out of Scope exists to prevent scope creep — list items that have been explicitly discussed and deferred.
- User stories use the canonical format: `As a [user type], I want to [action] so that [outcome].`
- Acceptance criteria use Given/When/Then language.
- The spec is **implementation-agnostic** — no API paths, schema field names, or service names. Those belong in the Design document.
- Open questions are resolved before the team moves to Design. Discovery may uncover answers.

---

## Discovery Document (`discovery.mdx`)

The discovery document answers: **What already exists in the codebase that is relevant to this feature?**

Each service section (Core, ML, App, Infrastructure) is populated by dispatching an agent with the appropriate service skill. The agent reviews the specification and reports:

- What entities, endpoints, components, and infrastructure already exist
- What can be reused without modification
- What constraints apply
- What gaps remain

Discovery findings directly inform the Design document — they determine which items are "new" versus "modify existing."

---

## Design Document (`design.mdx`)

The design document answers: **How does the system work?**

### Required Sections

| Section | Purpose |
|---------|---------|
| **System Context** | C4 Level 1 — service topology with a Mermaid diagram |
| **Data Flow** | Sequence diagrams for each primary user flow |
| **Architecture Pattern Justification** | Why this pattern was chosen (positive statement) |
| **API Design** | New and modified endpoints, request/response shapes, WebSocket events |
| **Data Schemas** | New tables, modified columns, migration SQL |
| **State Management** | SWR cache keys, optimistic update hooks, real-time subscriptions |
| **Rejected Alternatives** | Concise table of considered approaches and why they were not chosen |

### Data Flow Diagrams

Each significant user flow gets its own named sequence diagram. Name flows descriptively:

- ✅ `Flow 1: Start Recording`
- ✅ `Flow 3a: Live Talking Points (Per Finalised Segment)`
- ❌ `Flow 1`, `Diagram A`, `Sequence 3`

Diagrams use `autonumber` to make steps referenceable in review comments.

### Architecture Pattern Justification

The justification table states the decision for each architectural criterion. It does not contain "Why not X?" rows — those belong in Rejected Alternatives.

```markdown
| Criterion | Decision |
|-----------|----------|
| **Data architecture** | Optimistic Mutation with Echo-Suppressed Streaming |
| **Transport** | WebSocket (bidirectional: commands upstream, events downstream) |
```

### Streaming Cadences Table

For features with real-time data flows, include a streaming cadences table that summarises update frequency and delivery path for each data type:

```markdown
| Data Type | Update Frequency | Delivery Path |
|-----------|-----------------|---------------|
| Transcript segments | Every segment | ML → HTTP stream → Core → WS → App |
| Tasks (system) | ~60 second batches | ML → HTTP stream → Core → WS → App |
```

---

## Implementation Document (`implementation.mdx`)

The implementation document answers: **Who builds what, in what order?**

### Delivery Rules (Mandatory)

Every implementation document must include these four rules verbatim:

1. **One phase = one branch = one PR.** Each phase is implemented on a dedicated branch and merged via a reviewed PR before the next phase starts.
2. **Phases are strictly sequential** unless the dependency column explicitly marks them as parallelisable.
3. **Run `./dev gen all` after any phase that changes Core or ML APIs** — the generation pipeline must complete before downstream phases (especially App) can consume new types.
4. **Every PR goes through `/code-review`** with the relevant service skill loaded.

### Phase Structure

Each phase specifies:
- **Agent** — the skill to dispatch (`core-engineer`, `ml-engineer`, `app-engineer`, `platform-engineer`)
- **Estimated Complexity** — S / M / L
- **Tasks** — concrete, actionable checkbox items
- **Definition of Done** — includes the exact `./dev test` command and code review requirement

### Integration Verification

After all phases merge, a cross-service verification step runs:

```markdown
- [ ] System tests pass (`./dev test system`)
- [ ] Smoke tests pass (`./dev test smoke`)
- [ ] Manual walkthrough of primary user stories from the specification
```

---

## Validation Document (`validation.mdx`)

The validation document answers: **How do we know the feature is correct?**

Test cases map acceptance criteria from the specification to specific test types (Unit, Service, System, E2E). Each phase's Definition of Done references this document for its review criteria.

The Rollback Plan describes the exact steps to undo the feature if production issues arise — not a generic statement, but a concrete sequence for each affected layer (database migration rollback, feature flag kill switch, API versioning).

---

## Lifecycle Progression

A feature moves through phases in order. No phase begins until the previous document is complete and agreed:

```
Specify → Discover → Design → Implement → Validate
```

- **Specify** is complete when all open questions are resolved and the spec is approved.
- **Discover** is complete when all service sections are populated and constraints are documented.
- **Design** is complete when all data flows are diagrammed, API and schema changes are defined, and Rejected Alternatives are documented.
- **Implement** proceeds one phase at a time, with a merged PR as the exit criterion for each phase.
- **Validate** is complete when all test cases pass and the rollback plan has been reviewed.
