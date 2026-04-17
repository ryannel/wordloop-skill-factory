# Strategic Roadmapping & Debt Management

Technical debt and documentation debt are strategic levers, not reactive obligations. Managing debt intentionally — with explicit budgets, friction metrics, and prioritized remediation — transforms it from a silent liability into a visible, controllable cost.

---

## Table of Contents
1. [Dynamic Debt Awareness](#dynamic-debt-awareness)
2. [Friction Metrics](#friction-metrics)
3. [Strategic Themes](#strategic-themes)
4. [Now/Next/Later Bucketing](#nownextlater-bucketing)
5. [Budgeted Remediation](#budgeted-remediation)
6. [Debt Inventory Template](#debt-inventory-template)
7. [Integration with ADRs](#integration-with-adrs)

---

## Dynamic Debt Awareness

Debt awareness is not a quarterly audit — it is a continuous, instrumented practice. Use metadata agents and automated analysis to continuously map system relationships and track where documentation friction accumulates.

### Signals to Monitor

| Signal | What It Reveals | Collection Method |
|--------|----------------|-------------------|
| **Support ticket frequency by topic** | Which documentation areas are failing (users ask support when docs don't answer their question) | Tag and categorize support tickets by docs section |
| **Search queries with no results** | What users are looking for that doesn't exist in the docs | Search analytics (Algolia, Typesense) |
| **Page bounce rate** | Which pages are not answering the user's question (they leave immediately) | Analytics (Plausible, GA) |
| **Time-on-page vs. page length** | Whether users are reading or just scanning/leaving | Analytics correlation |
| **PR age for docs changes** | Whether documentation updates are bottlenecked in review | Git metrics |
| **Diagram age** | Which diagrams have not been updated relative to code changes | Git log comparison (last commit to diagram vs. last commit to related source files) |

### Automated Staleness Detection

Implement a CI job that detects stale documentation by comparing documentation file modification dates against related source code modification dates:

```bash
#!/bin/bash
# Detect documentation that may be stale relative to source code
STALE_THRESHOLD_DAYS=90

for doc in docs/**/*.md; do
  doc_date=$(git log -1 --format="%ct" -- "$doc")
  related_src=$(grep -l "$(basename $doc .md)" src/**/* 2>/dev/null | head -1)

  if [ -n "$related_src" ]; then
    src_date=$(git log -1 --format="%ct" -- "$related_src")
    age_diff=$(( (src_date - doc_date) / 86400 ))

    if [ $age_diff -gt $STALE_THRESHOLD_DAYS ]; then
      echo "STALE: $doc (source changed ${age_diff} days after doc)"
    fi
  fi
done
```

---

## Friction Metrics

### Mean Time to Change (MTTC)

MTTC measures how long it takes to make a change to the documentation from the moment the need is identified to the moment the change is live. High MTTC indicates process friction — review bottlenecks, complex build pipelines, or unclear ownership.

**Target:** Documentation changes should ship within 24 hours of being identified. If MTTC exceeds 72 hours, investigate the bottleneck.

### Mean Time to First Value (MTTFV)

MTTFV measures how long it takes a new user to achieve their first meaningful outcome using the documentation. This is the documentation equivalent of "time-to-first-call" for API docs.

**Measurement:** Track the time from a user's first page view to their first successful interaction (API call, deployment, configuration change). This requires instrumentation in both the docs site and the product.

### Documentation Coverage Ratio

The ratio of documented features to total features. A coverage ratio below 80% indicates undocumented capabilities that create support burden and reduce user confidence.

**Calculation:** `(Features with documentation) / (Total features) × 100`

---

## Strategic Themes

Group documentation work into strategic themes that align with product and engineering objectives. Themes prevent documentation work from devolving into an unstructured queue of ad-hoc requests.

### Example Themes

| Theme | Description | Documentation Work Included |
|-------|-------------|----------------------------|
| **Developer Onboarding** | Reduce time-to-first-value for new API consumers | Quickstarts, tutorials, interactive playground, onboarding checklists |
| **Ecosystem Reliability** | Increase confidence in SDK and API documentation accuracy | Automated SDK generation, contract tests, Vale linting in CI |
| **AI Readiness** | Optimize documentation for AI agent consumption | llms.txt implementation, MCP server, semantic markup improvements |
| **Architectural Clarity** | Ensure all services have current architecture documentation | C4 diagrams, ADR backlog, drift detection automation |
| **Debt Remediation** | Pay down accumulated documentation debt | Stale page updates, broken link fixes, diagram refreshes |

### Theme Prioritization

Prioritize themes by their impact on key metrics:

```
Impact Score = (MTTFV Improvement × User Volume) + (Support Ticket Reduction × Cost per Ticket)
```

Themes with the highest impact score receive the highest priority. Reassess quarterly.

---

## Now/Next/Later Bucketing

The Now/Next/Later model provides a simple prioritization framework that balances urgency with strategic value.

### Definition

| Bucket | Time Horizon | Content | Commitment Level |
|--------|-------------|---------|-----------------|
| **Now** | This sprint / iteration | Items actively being worked on | Committed — these will be completed |
| **Next** | Next 1-2 sprints | Items with clear scope, ready to start | Planned — scope is defined, not yet started |
| **Later** | Beyond 2 sprints | Items identified but not yet scoped | Identified — may change or be deprioritized |

### Applying to Documentation Debt

1. **Inventory the debt:** List all known documentation problems (stale pages, missing guides, outdated diagrams, absent ADRs).
2. **Score each item:** Use the friction metrics (support tickets, search failures, page bounces) to assign an impact score.
3. **Bucket by impact and urgency:**
   - **Now:** High-impact items that are blocking users or creating support load.
   - **Next:** Medium-impact items that improve experience but are not blocking.
   - **Later:** Low-impact items or items that require upstream changes (e.g., an API redesign) before documentation can be updated.
4. **Reassess every sprint:** Move items between buckets as priorities shift.

### Example

```markdown
## Documentation Debt Board

### Now (Sprint 14)
- [ ] Update authentication guide to reflect OAuth 2.1 migration (ADR-005)
- [ ] Add missing error codes to the API reference
- [ ] Fix 12 broken cross-reference links in the SDK guide

### Next (Sprint 15-16)
- [ ] Create interactive onboarding tutorial for the new CLI tool
- [ ] Add C4 Container diagram for the notification service
- [ ] Implement llms.txt for the documentation site

### Later
- [ ] Write Debt-Aware ADR for the planned database migration
- [ ] Create video walkthroughs for top 5 tutorials
- [ ] Implement MCP documentation server
```

---

## Budgeted Remediation

Documentation debt, like technical debt, requires explicit budget allocation. Without a budget, debt remediation competes with (and loses to) feature work in every sprint.

### The 15-20% Rule

Earmark **15-20% of sprint capacity** for documentation debt remediation. This is not a suggestion — it is a structural commitment that ensures debt does not compound silently.

### Allocation Strategy

| Allocation | Target | Example |
|------------|--------|---------|
| **5%** | Reactive fixes | Broken links, typos, factual corrections reported by users |
| **5%** | Staleness remediation | Updating pages identified by the automated staleness detection |
| **5-10%** | Strategic debt paydown | Addressing items from the Now bucket of the debt board |

### Tracking

Track remediation velocity as a team metric:

- **Debt items closed per sprint:** Are you paying down debt faster than you are accumulating it?
- **Debt age:** How old are the oldest unresolved debt items? Items older than 6 months are compounding.
- **Coverage ratio trend:** Is the documentation coverage ratio improving or declining over time?

---

## Debt Inventory Template

```markdown
# Documentation Debt Inventory

| ID | Description | Type | Impact | Bucket | Related ADR | Age |
|----|-------------|------|--------|--------|-------------|-----|
| D-001 | Authentication guide is outdated post-OAuth migration | Documentation | High | Now | ADR-005 | 3 weeks |
| D-002 | Container diagram missing notification service | Documentation | Medium | Next | — | 2 months |
| D-003 | No llms.txt file for AI agent discovery | Documentation | Medium | Next | — | — |
| D-004 | 14 deprecated endpoints still in API reference | Code | High | Now | — | 1 month |
| D-005 | CLI reference has no usage examples | Documentation | Low | Later | — | 4 months |
```

---

## Integration with ADRs

Every debt item with structural impact should be linked to an ADR:

- **Debt created by an ADR:** The ADR's Technical Debt Assessment section documents the expected debt. Create a corresponding item in the debt inventory.
- **Debt requiring a new decision:** When remediation requires an architectural change (e.g., migrating to a new documentation platform), create a new ADR to record the decision.
- **Debt resolved by an ADR:** When an ADR supersedes a previous decision, close the corresponding debt items from the original ADR.

This bidirectional linking between ADRs and the debt inventory ensures that decisions and their consequences are tracked together, preventing the common failure mode where debt is documented but never connected to the decision that created it.
