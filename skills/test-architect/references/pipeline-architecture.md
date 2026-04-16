# Pipeline Architecture

This module defines how all test types, quality gates, and validation signals compose into a unified CI/CD pipeline. Individual modules cover specific tools and techniques — this module shows how they fit together.

---

## Table of Contents

1. [The Four Pipeline Stages](#the-four-pipeline-stages)
2. [PR Pipeline](#pr-pipeline)
3. [Merge Pipeline](#merge-pipeline)
4. [Manual Validation](#manual-validation)
5. [Scheduled Pipeline](#scheduled-pipeline)
6. [Gate Composition](#gate-composition)
7. [Pipeline Performance Budgets](#pipeline-performance-budgets)

---

## The Four Pipeline Stages

```
┌─────────────────────────────────────────────────────────────────────┐
│ 1. PR PIPELINE (Every Commit)                    Budget: < 10 min  │
│    ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐            │
│    │ Small    │ │ Service  │ │ SAST/SCA │ │ Incremental│           │
│    │ Tests    │ │ Tests    │ │ Security │ │ Mutation  │            │
│    └──────────┘ └──────────┘ └──────────┘ └──────────┘            │
│    Gate: ALL must pass to merge                                    │
├─────────────────────────────────────────────────────────────────────┤
│ 2. MERGE PIPELINE (Trunk)                        Budget: < 20 min  │
│    ┌──────────┐ ┌──────────┐ ┌──────────┐                         │
│    │ System   │ │ CDCT     │ │ can-i-   │                         │
│    │ Tests    │ │ Verify   │ │ deploy   │                         │
│    └──────────┘ └──────────┘ └──────────┘                         │
│    Gate: ALL must pass to deploy                                   │
├─────────────────────────────────────────────────────────────────────┤
│ 3. MANUAL (Developer-Initiated)                  Budget: flexible  │
│    ┌──────────┐ ┌──────────┐                                      │
│    │ Live     │ │ Live     │                                      │
│    │ Service  │ │ System   │                                      │
│    │ Tests    │ │ Tests    │                                      │
│    └──────────┘ └──────────┘                                      │
│    Gate: NONE — developer judgment                                 │
├─────────────────────────────────────────────────────────────────────┤
│ 4. SCHEDULED (Nightly)                           Budget: < 60 min  │
│    ┌──────────┐ ┌──────────┐ ┌──────────┐                         │
│    │ Full     │ │ Chaos    │ │ eBPF     │                         │
│    │ Mutation │ │ Experiments│ │ Profiling│                        │
│    └──────────┘ └──────────┘ └──────────┘                         │
│    Gate: Alert on regression, do not block                         │
└─────────────────────────────────────────────────────────────────────┘
```

---

## PR Pipeline

**Trigger:** Every push to a pull request branch.
**Budget:** < 10 minutes total.
**Gate:** All checks must pass before merge is allowed.

### Stage Composition

```yaml
# .github/workflows/pr.yml
name: PR Pipeline
on: pull_request

jobs:
  # ── Stage 1: Parallel quality checks ──────────────────
  
  small-tests:
    name: Small Tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: go test -short ./...           # Go
      - run: pytest tests/unit/ -x          # Python
    # Budget: < 30 seconds

  service-tests:
    name: Service Tests
    runs-on: ubuntu-latest
    services:
      docker:
        image: docker:dind
    steps:
      - uses: actions/checkout@v4
      - run: go test ./internal/... -v -timeout 5m
      - run: pytest tests/service/ -v --timeout=300
    # Budget: < 5 minutes (Testcontainers startup + tests)

  security:
    name: Security Scanning
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: gosec ./...
      - run: govulncheck ./...
      - run: bandit -r src/
      - run: pip-audit
    # Budget: < 2 minutes

  incremental-mutation:
    name: Mutation Testing (Changed Files)
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - run: |
          CHANGED=$(git diff --name-only origin/main -- '*.go' | grep -v _test.go)
          if [ -n "$CHANGED" ]; then
            go-mutesting --diff origin/main ./...
          fi
    # Budget: < 3 minutes (incremental only)
```

### What Each Stage Validates

| Stage | What It Catches | Module Reference |
|-------|----------------|-----------------|
| Small Tests | Logic errors in pure functions | `taxonomy-and-naming.md` |
| Service Tests | API behavior, DB correctness, event emission, auth | `validation-layers.md`, `emulation-strategy.md` |
| Security Scanning | CVEs, injection vulnerabilities, hardcoded secrets | `security-testing.md` |
| Incremental Mutation | Weak assertions in changed code | `mutation-testing.md` |

---

## Merge Pipeline

**Trigger:** Merge to trunk / main branch.
**Budget:** < 20 minutes total.
**Gate:** All checks must pass before deployment is allowed.

### Stage Composition

```yaml
# .github/workflows/merge.yml
name: Merge Pipeline
on:
  push:
    branches: [main]

jobs:
  system-tests:
    name: System Tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: docker compose -f docker-compose.test.yml up -d
      - run: ./scripts/wait-for-healthy.sh --timeout 120
      - run: go test ./tests/system/... -v -timeout 10m
      - run: ./scripts/collect-traces.sh
      - if: always()
        run: docker compose -f docker-compose.test.yml down -v
    # Budget: < 15 minutes

  contract-verification:
    name: CDCT Verification
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: go test ./contracts/provider/... -v
      - run: |
          pact-broker can-i-deploy \
            --pacticipant wordloop-core \
            --version ${{ github.sha }} \
            --to-environment production
    # Budget: < 5 minutes
```

### What Each Stage Validates

| Stage | What It Catches | Module Reference |
|-------|----------------|-----------------|
| System Tests | Cross-service failures, broken traces, event propagation | `validation-layers.md`, `trace-validation.md` |
| CDCT Verification | Contract violations between services | `cdct-implementation.md` |

---

## Manual Validation

**Trigger:** Developer runs locally or via `workflow_dispatch`.
**Budget:** At the developer's discretion.
**Gate:** No automated gate — developer uses judgment.

### When Developers Run Live Tests

```
Developer working on OpenAI integration
  │
  ├── Iterating on prompt engineering
  │   └── Run Live Service Tests repeatedly (< 30s each)
  │       └── `go test ./tests/live/... -run TestOpenAI -v`
  │
  └── Integration code is ready, want holistic confidence
      └── Run Live System Tests once (< 15 min)
          └── `go test ./tests/live-system/... -v`
```

### GitHub Actions Manual Trigger

```yaml
# .github/workflows/live-tests.yml
name: Live Tests
on:
  workflow_dispatch:
    inputs:
      scope:
        description: 'Which live tests to run'
        type: choice
        options:
          - live-service
          - live-system
      
jobs:
  live-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: |
          if [ "${{ inputs.scope }}" = "live-service" ]; then
            go test ./tests/live/... -v -timeout 5m
          else
            docker compose -f docker-compose.test.yml up -d
            go test ./tests/live-system/... -v -timeout 20m
            docker compose -f docker-compose.test.yml down -v
          fi
    env:
      OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
      GOOGLE_OAUTH_CLIENT_ID: ${{ secrets.GOOGLE_OAUTH_CLIENT_ID }}
```

---

## Scheduled Pipeline

**Trigger:** Nightly cron schedule.
**Budget:** < 60 minutes.
**Gate:** Alert on regression, do not block.

### Stage Composition

```yaml
# .github/workflows/nightly.yml
name: Nightly Quality
on:
  schedule:
    - cron: '0 2 * * *'  # 2 AM UTC

jobs:
  full-mutation:
    name: Full Mutation Testing
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: |
          go-mutesting ./internal/core/service/... \
            --parallel $(nproc) \
            --score-threshold 0.75
      - run: |
          mutmut run --paths-to-mutate=src/
          mutmut results
    # Budget: < 30 minutes

  chaos:
    name: Chaos Experiments
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: docker compose -f docker-compose.test.yml up -d
      - run: ./scripts/wait-for-healthy.sh --timeout 120
      - run: go test ./tests/chaos/... -v -timeout 15m
      - if: always()
        run: docker compose -f docker-compose.test.yml down -v
    # Budget: < 20 minutes

  profiling:
    name: Performance Profiling
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: docker compose -f docker-compose.test.yml up -d
      - run: go test ./tests/system/... -v -bench=. -benchmem
      - run: ./scripts/compare-profiles.sh --baseline main
      - if: always()
        run: docker compose -f docker-compose.test.yml down -v
    # Budget: < 10 minutes
```

---

## Gate Composition

### How Gates Interact

```
PR Pipeline pass? ──────► Merge allowed
                              │
                     Merge Pipeline pass? ──────► Deploy allowed
                                                       │
                                              can-i-deploy pass? ──► Production
```

### Gate Severity Matrix

| Signal | PR Gate | Merge Gate | Nightly |
|--------|---------|-----------|---------|
| Small test failure | **Block** | **Block** | Alert |
| Service test failure | **Block** | **Block** | Alert |
| System test failure | N/A | **Block** | Alert |
| Security critical/high | **Block** | **Block** | Alert |
| Security medium/low | Warn | Warn | Log |
| CDCT contract violation | N/A | **Block** | Alert |
| Mutation score regression | Warn | Warn | **Alert + ticket** |
| Chaos experiment failure | N/A | N/A | **Alert + ticket** |
| Performance regression | N/A | N/A | **Alert + ticket** |

---

## Pipeline Performance Budgets

### Why Budgets Matter

Pipeline speed directly correlates with developer productivity. Every minute added to the PR pipeline is a minute of context-switching for every developer on every push. Protect these budgets aggressively.

### Budget Targets

| Pipeline | Target | Hard Limit | If Exceeded |
|----------|--------|-----------|-------------|
| PR Pipeline | < 5 min | 10 min | Investigate immediately. Parallelise or defer slow checks. |
| Merge Pipeline | < 10 min | 20 min | Move expensive checks to nightly. Optimise System Test containers. |
| Manual (Live Service) | < 30s/test | 2 min/test | Acceptable — developer controls when to run. |
| Manual (Live System) | < 15 min | 30 min | Acceptable for full-cluster validation. |
| Nightly | < 30 min | 60 min | Parallelise mutation and chaos. Use incremental mutation. |

### Optimisation Strategies

| Strategy | Impact |
|----------|--------|
| **Testcontainers singleton pattern** | Eliminates container restart between tests within a suite |
| **Parallel test execution** | Go: `-parallel N`, pytest: `-n auto` |
| **Incremental mutation** | Only mutate files changed since last run |
| **Docker layer caching** | Cache test images in CI to avoid rebuilding |
| **Test splitting by size** | Run Small tests first (fastest feedback), Service tests in parallel |

---

## See Also

- `validation-layers.md` — Test types and their execution models
- `risk-and-chaos.md` — Chaos experiments in the Nightly stage
- `mutation-testing.md` — Incremental mutation in PR pipeline
