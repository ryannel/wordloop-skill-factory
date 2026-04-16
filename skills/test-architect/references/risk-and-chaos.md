# Risk-Based Engineering & Chaos Testing

This module defines how to map test coverage to business impact and how to implement chaos engineering as a validation discipline within the CI/CD pipeline.

---

## Table of Contents

1. [Risk-Based Coverage Mapping](#risk-based-coverage-mapping)
2. [The Risk Matrix](#the-risk-matrix)
3. [Risk Score to Test Depth Mapping](#risk-score-to-test-depth-mapping)
4. [Chaos Engineering in CI/CD](#chaos-engineering-in-cicd)
5. [Fault Injection Patterns](#fault-injection-patterns)
6. [Tooling Reference](#tooling-reference)
7. [Pipeline Integration](#pipeline-integration)

---

## Risk-Based Coverage Mapping

Every test must justify its existence through a measurable risk signal. The question is never "do we have enough test coverage?" but "are we testing the things that matter?"

### Why Not 100% Code Coverage?

Line coverage is a vanity metric. A test suite can have 100% coverage and still fail to catch critical bugs if:

- Assertions are weak or absent (tests that execute code but verify nothing)
- Edge cases and boundary conditions are untested
- Integration points are mocked away
- Error paths are exercised but not validated

Use **Risk-Based Coverage** of the Critical Path combined with **Mutation Testing** to prove assertion effectiveness.

---

## The Risk Matrix

Score every module across three parameters on a **1–5 scale**:

```
Risk Score = Impact × Technical Complexity × Historical Failure Rate
```

### Parameter Definitions

| Parameter | 1 (Low) | 3 (Medium) | 5 (High) |
|-----------|---------|------------|----------|
| **Impact** | Internal tooling, non-customer-facing | Affects user experience indirectly | Payment processing, data integrity, auth |
| **Technical Complexity** | Simple CRUD, no external deps | Multiple integrations, async flows | Distributed transactions, ML inference, real-time |
| **Historical Failure Rate** | No incidents in 6+ months | Occasional bugs, stable area | Frequent regressions, known fragile area |

### Score Ranges

| Score Range | Risk Level | Recommended Action |
|-------------|------------|-------------------|
| **1–5** | Low | Small tests + SAST/SCA scans only. Minimal investment. |
| **6–14** | Medium | Service Tests + contract tests. Standard coverage. |
| **15–25** | High | Full Service Test + System Test coverage + Live System Tests + chaos experiments + exploratory testing sessions. |

### Example Risk Assessment

| Module | Impact | Complexity | Failure Rate | Score | Risk Level |
|--------|--------|------------|-------------|-------|------------|
| User authentication | 5 | 4 | 3 | 60 → capped at 25 | **High** |
| Meeting creation API | 4 | 3 | 2 | 24 | **High** |
| Audio transcription pipeline | 5 | 5 | 4 | 100 → capped at 25 | **High** |
| Admin settings page | 2 | 1 | 1 | 2 | **Low** |
| Static content API | 1 | 1 | 1 | 1 | **Low** |

> **Note:** Scores above 25 are capped at 25 for classification purposes, but the raw score is useful for prioritization within the "High" tier.

---

## Risk Score to Test Depth Mapping

| Risk Level | Required Test Layers | Additional Requirements |
|------------|---------------------|------------------------|
| **Low (1–5)** | Small tests only | SAST/SCA scanning in PR pipeline |
| **Medium (6–14)** | Service Tests | CDCT contracts, mutation testing on critical assertions |
| **High (15–25)** | Service Tests + System Tests | Live System Tests on release branches, chaos experiments, exploratory sessions, eBPF profiling |

---

## Chaos Engineering in CI/CD

Chaos engineering proves that resilience mechanisms (circuit breakers, retries, graceful degradation) actually work — not by hoping they work in production, but by deliberately breaking things during System Testing.

### Core Methodology

1. **Define Steady State:** Establish measurable health indicators (latency, error rate, throughput) that represent normal operation.
2. **Hypothesize:** "If we kill Service B's database, Service A should degrade gracefully and return cached results."
3. **Inject Fault:** Introduce the failure condition into the test environment.
4. **Observe:** Measure actual behavior against the hypothesis.
5. **Verdict:** If the system recovers within SLO bounds, the test passes. If not, it reveals a real resilience gap.

### Fault Categories

| Category | Examples | What It Validates |
|----------|----------|-------------------|
| **Network Faults** | Latency injection, packet loss, connection timeout, DNS failure | Circuit breakers, timeout configuration, retry logic |
| **Service Faults** | Container kill, process crash, OOM kill | Graceful degradation, health check recovery, pod restart |
| **Resource Faults** | CPU throttling, memory pressure, disk full | Backpressure mechanisms, resource limits, autoscaling |
| **Dependency Faults** | Database unavailability, cache eviction, message broker partition | Fallback strategies, stale cache serving, dead letter queues |

---

## Fault Injection Patterns

### Pattern 1: Toxiproxy for Network Faults

**Toxiproxy** sits between your service and its dependency, introducing network-level faults programmatically.

```go
func TestCreateMeeting_ShouldReturnCached_WhenDatabaseTimesOut(t *testing.T) {
    // Setup: Toxiproxy sits between service and PostgreSQL
    proxy := toxiproxy.NewProxy("postgres", "postgres:5432")
    
    // Seed valid data
    seedMeeting(t, db, "meeting_001")
    
    // Inject fault: 5-second latency on all database calls
    proxy.AddToxic("latency", "latency", "downstream", 1.0, toxiproxy.Attributes{
        "latency": 5000,
    })
    
    // Execute: service should timeout and return cached result
    resp := httpGet(t, "/api/v1/meetings/meeting_001")
    assert.Equal(t, 200, resp.StatusCode) // graceful degradation
    
    // Verify: response came from cache, not database
    assert.Equal(t, "cached", resp.Header.Get("Data-Source"))
    
    // Cleanup
    proxy.RemoveToxic("latency")
}
```

### Pattern 2: Container Lifecycle for Service Faults

Kill or pause containers during test execution to simulate pod evictions:

```go
func TestEventPropagation_ShouldRecover_WhenMLServiceRestarted(t *testing.T) {
    // Verify steady state: event propagation works
    createMeeting(t)
    assertEventReceived(t, "meeting.created")
    
    // Inject fault: kill the ML service container
    err := mlContainer.Stop(ctx, nil)
    require.NoError(t, err)
    
    // Verify degraded state: core service handles ML unavailability
    resp := createMeeting(t)
    assert.Equal(t, 201, resp.StatusCode) // core still works
    
    // Recovery: restart ML service
    err = mlContainer.Start(ctx)
    require.NoError(t, err)
    waitForHealthy(t, mlContainer)
    
    // Verify recovery: events propagate again
    createMeeting(t)
    assertEventReceived(t, "meeting.created")
}
```

### Pattern 3: Kubernetes-Native Chaos (System Tests)

For tests running in ephemeral Kubernetes clusters (kind, k3s), use Chaos Mesh or LitmusChaos:

```yaml
# chaos-mesh/network-partition.yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: partition-ml-from-core
spec:
  action: partition
  mode: all
  selector:
    labelSelectors:
      app: wordloop-ml
  direction: both
  target:
    selector:
      labelSelectors:
        app: wordloop-core
  duration: "30s"
```

---

## Tooling Reference

| Tool | Type | Best For |
|------|------|----------|
| **Toxiproxy** | Network proxy | Fine-grained network fault injection between specific services. Programmable via API. |
| **Chaos Mesh** | Kubernetes operator | Pod faults, network partitions, I/O chaos, time skew in K8s environments. |
| **LitmusChaos** | Kubernetes operator | Pre-built chaos experiments with observability integrations. |
| **tc / netem** | Linux kernel | Raw network emulation (latency, loss, corruption) at the kernel level. |
| **stress-ng** | System stress | CPU, memory, and I/O stress testing for resource exhaustion scenarios. |

---

## Pipeline Integration

### Resilience Gate in CI/CD

```yaml
# .github/workflows/system-tests.yml
jobs:
  system-test:
    steps:
      - name: Deploy Ephemeral Cluster
        run: docker compose -f docker-compose.test.yml up -d
        
      - name: Wait for Steady State
        run: ./scripts/wait-for-healthy.sh --timeout 120
        
      - name: Run System Tests
        run: go test ./tests/system/... -v -timeout 10m
        
      - name: Run Chaos Experiments
        run: go test ./tests/chaos/... -v -timeout 5m
        
      - name: Collect Traces & Metrics
        if: always()
        run: ./scripts/collect-observability.sh
        
      - name: Teardown
        if: always()
        run: docker compose -f docker-compose.test.yml down -v
```

### Budget and Frequency

| Experiment Type | Frequency | Duration Budget |
|----------------|-----------|-----------------|
| Network latency injection | Every merge to trunk | < 2 minutes |
| Service kill/restart | Every merge to trunk | < 3 minutes |
| Full partition scenarios | Nightly schedule | < 10 minutes |
| Resource exhaustion | Weekly schedule | < 15 minutes |

---

## See Also

- `emulation-strategy.md` — Toxiproxy and container lifecycle patterns
- `validation-layers.md` — Risk score to test type mapping
- `pipeline-architecture.md` — Chaos experiments in the Nightly pipeline
