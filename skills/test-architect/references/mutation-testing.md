# Mutation Testing

This module defines how to measure test suite effectiveness through systematic fault injection, replacing arbitrary code coverage targets with mathematical proof that assertions evaluate correct logical paths.

---

## Table of Contents

1. [Core Principle](#core-principle)
2. [How Mutation Testing Works](#how-mutation-testing-works)
3. [Mutation Operators](#mutation-operators)
4. [Tooling by Language](#tooling-by-language)
5. [CI/CD Integration](#cicd-integration)
6. [Handling Equivalent Mutants](#handling-equivalent-mutants)
7. [Quality Gates](#quality-gates)
8. [Best Practices](#best-practices)

---

## Core Principle

Code coverage tells you which lines were **executed**. Mutation testing tells you which lines were **actually validated**.

A test suite with 90% line coverage might still miss critical bugs if its assertions are weak, absent, or testing the wrong things. Mutation testing injects small, synthetic faults into production code and checks whether the test suite catches them. If a mutant survives (tests still pass despite the fault), it reveals a gap in assertion quality.

This is the only metric that provides mathematical proof that your tests evaluate correct logical paths rather than merely executing them.

---

## How Mutation Testing Works

```
1. Mutation engine modifies production code
   (e.g., changes `>` to `>=`, removes a function call, replaces `true` with `false`)

2. Test suite runs against the mutated code

3. Outcomes:
   ├── Test FAILS → Mutant "killed" ✓ (your test caught the bug)
   └── Test PASSES → Mutant "survived" ✗ (your test missed the bug)

4. Mutation Score = killed / (killed + survived)
```

### Mutation Score Interpretation

| Score | Interpretation |
|-------|---------------|
| **> 80%** | Strong test suite. Most logical paths are properly validated. |
| **60–80%** | Moderate. Assertions exist but have gaps. Focus on surviving mutants in high-risk code. |
| **< 60%** | Weak. Many assertions are either missing, testing wrong behavior, or overly generic. |

---

## Mutation Operators

Mutation operators define the types of faults injected. Common operators include:

| Operator | Original | Mutated | What It Tests |
|----------|----------|---------|---------------|
| **Relational** | `x > 0` | `x >= 0` | Boundary condition assertions |
| **Arithmetic** | `a + b` | `a - b` | Calculation correctness |
| **Boolean** | `if enabled` | `if !enabled` | Conditional logic branches |
| **Return Value** | `return result` | `return nil` | Error handling and null checks |
| **Void Call** | `notify(user)` | _(removed)_ | Side effect validation |
| **Constant** | `timeout = 30` | `timeout = 0` | Configuration value assertions |
| **Negation** | `x != y` | `x == y` | Equality check correctness |

---

## Tooling by Language

### Go: gomu

```bash
# Install
go install github.com/avito-tech/go-mutesting/cmd/go-mutesting@latest

# Run against specific packages
go-mutesting ./internal/core/service/...

# Incremental mode: only mutate files changed since main
go-mutesting --diff main ./...
```

**Features:**
- Incremental analysis via Git diff (only mutate changed files)
- Parallel execution across multiple cores
- Integration with `go test` — no special test framework required

### Python: mutmut

```bash
# Install
pip install mutmut

# Run against source directory
mutmut run --paths-to-mutate=src/

# View results
mutmut results

# Show specific surviving mutant
mutmut show 42
```

**Features:**
- Integrates with pytest
- Caches results for incremental re-runs
- HTML report generation

### Configuration Example (mutmut)

```ini
# setup.cfg
[mutmut]
paths_to_mutate=src/
tests_dir=tests/
runner=python -m pytest -x --timeout=10
dict_synonyms=Struct,NamedTuple
```

---

## CI/CD Integration

### Strategy: Selective, Scheduled Execution

Mutation testing is compute-intensive. Running it on every commit is impractical for large codebases. Use a tiered strategy:

| Trigger | Scope | Purpose |
|---------|-------|---------|
| **PR (on critical paths)** | Only files changed in the PR that touch high-risk modules | Fast feedback on new code quality |
| **Nightly Schedule** | Full mutation run on high-risk packages | Comprehensive quality assessment |
| **Release Gate** | Core business logic packages | Pre-release quality validation |

### GitHub Actions Example

```yaml
# .github/workflows/mutation-testing.yml
name: Mutation Testing
on:
  schedule:
    - cron: '0 2 * * *'  # Nightly at 2 AM
  workflow_dispatch:       # Manual trigger

jobs:
  mutate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Run Mutation Tests (Go)
        run: |
          go install github.com/avito-tech/go-mutesting/cmd/go-mutesting@latest
          go-mutesting ./internal/core/service/... \
            --parallel $(nproc) \
            --score-threshold 0.75
        
      - name: Upload Report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: mutation-report
          path: mutation-report.html
```

### PR-Scoped Incremental Run

```yaml
# Only mutate files changed in this PR
- name: Incremental Mutation Test
  run: |
    CHANGED_FILES=$(git diff --name-only origin/main -- '*.go' | grep -v '_test.go')
    if [ -n "$CHANGED_FILES" ]; then
      go-mutesting --diff origin/main ./...
    fi
```

---

## Handling Equivalent Mutants

Some mutations do not change program behavior even though they modify source code. These are **equivalent mutants** and cannot be killed.

### Common Equivalent Mutant Patterns

| Pattern | Example | Why It's Equivalent |
|---------|---------|-------------------|
| Dead code mutation | Modifying unreachable branch | Code never executes regardless |
| Redundant condition | `x >= 0` when `x` is always positive | Condition already guaranteed by type/domain |
| Functionally identical | `a * 1` → `a * -1` where `a` is always 0 | Result is the same |

### How to Handle

1. **Don't obsess.** A small number of surviving equivalent mutants is normal and acceptable.
2. **Suppress known equivalents.** Most mutation tools support ignore annotations:
   ```go
   // mutmut: skip
   // go-mutesting: skip
   ```
3. **Investigate before suppressing.** Verify the mutant is truly equivalent, not a real gap. When in doubt, write the test.

---

## Quality Gates

### Recommended Thresholds

| Context | Minimum Mutation Score | Rationale |
|---------|----------------------|-----------|
| **New code (PR)** | 75% | New code should have strong assertions from the start |
| **Core business logic** | 80% | Critical paths demand high confidence |
| **Infrastructure/glue code** | 60% | Lower-risk code can tolerate more survivors |
| **Generated code (ORM, protobuf)** | Exclude | Mutating generated code tests the generator, not your logic |

### Gate Enforcement

```bash
# Fail CI if mutation score drops below threshold
go-mutesting ./internal/core/service/... --score-threshold 0.75
# Exit code 1 if score < 75%
```

---

## Best Practices

| Practice | Rationale |
|----------|-----------|
| **Run mutation testing after you have baseline coverage** | Mutation testing is most valuable when you already have tests that execute the code. Starting from 0% coverage, write tests first. |
| **Focus on surviving mutants in high-risk modules** | Not all survivors matter equally. Prioritize investigating survivors in modules with high Risk Matrix scores. |
| **Use mutation score trends, not absolute values** | Track whether the score improves over time. A declining score signals assertion quality degradation. |
| **Parallelize aggressively** | Mutation testing benefits enormously from parallel execution. Use all available CPU cores. |
| **Cache results** | Both mutmut and gomu support caching. Only re-test mutants in files that changed. |
| **Don't aim for 100% mutation score** | Equivalent mutants and diminishing returns make 100% impractical. 75–85% is a strong, sustainable target. |

---

## See Also

- `risk-and-chaos.md` — Risk Matrix determines where to focus mutation testing
- `pipeline-architecture.md` — Incremental vs full mutation in CI stages
- `ai-augmented-quality.md` — AI-assisted gap detection from surviving mutants
