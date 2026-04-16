# Shift-Right Testing: Validating in Production

This module defines strategies for treating production as an active segment of the testing lifecycle. Shift-right practices validate behavior under real-world load, state entropy, and unpredictable user patterns that no pre-production environment can fully replicate.

---

## Table of Contents

1. [Core Principle](#core-principle)
2. [Traffic Shadowing (Teeing)](#traffic-shadowing-teeing)
3. [Progressive Delivery](#progressive-delivery)
4. [Feature Flag Testing](#feature-flag-testing)
5. [Synthetic Monitoring](#synthetic-monitoring)
6. [Observability as Testing](#observability-as-testing)
7. [Guardrails](#guardrails)

---

## Core Principle

Pre-production testing validates **known scenarios**. Production testing validates **unknown scenarios** — the real-world payload entropy, traffic patterns, and state combinations that are impossible to predict or synthesize.

Shift-right testing is not a replacement for Service Tests and System Tests. It is an additive validation discipline that catches the "works in staging" paradox by bringing test instrumentation directly to the deployment boundary.

---

## Traffic Shadowing (Teeing)

### What It Is

Mirror live, **read-only** production traffic to a transient staging or canary deployment. The shadow environment processes the same requests as production but its responses are discarded — never reaching real users.

### Architecture

```
┌──────────────────────────────────────────────────┐
│ Production Load Balancer                         │
│  ├── Primary: Production Server (serves users)   │
│  └── Shadow: Canary Server (discards responses)  │
│              ↓                                    │
│       Compare: latency, errors, response diff    │
└──────────────────────────────────────────────────┘
```

### What Traffic Shadowing Validates

| Validation Target | Description |
|-------------------|-------------|
| **Performance under real load** | Does the new version handle the same traffic volume without degradation? |
| **Payload entropy** | Does the new code handle the full range of real-world data shapes, encoding quirks, and edge cases? |
| **Scaling behavior** | Does auto-scaling respond correctly to real traffic patterns? |
| **Resource consumption** | Does memory/CPU usage remain within budget under production-like conditions? |

### Implementation Patterns

**Istio Service Mesh:**
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
spec:
  hosts:
    - wordloop-core
  http:
    - route:
        - destination:
            host: wordloop-core
            subset: stable
          weight: 100
      mirror:
        host: wordloop-core
        subset: canary
      mirrorPercentage:
        value: 10.0  # Shadow 10% of traffic
```

**NGINX:**
```nginx
location /api/ {
    proxy_pass http://production-upstream;
    mirror /shadow;
    mirror_request_body on;
}

location = /shadow {
    internal;
    proxy_pass http://canary-upstream$request_uri;
}
```

### Constraints

- **Read-only traffic only.** Never shadow write operations (POST, PUT, DELETE) to a shadow that shares state with production. Use separate databases or discard writes.
- **Monitor shadow health separately.** Shadow failures should trigger alerts but not affect production serving.
- **Budget for double compute.** Shadowed traffic doubles the request processing load. Shadow only a percentage of traffic if cost is a concern.

---

## Progressive Delivery

### What It Is

Features behind deployment controls (canary releases, A/B flags) are subjected to automated CI that explicitly targets varying states on live systems. Instead of testing a feature in isolation, you test the **transition** from off to on and the **coexistence** of old and new code paths.

### Canary Releases

```
Deploy v2 to 5% of pods
  → Monitor error rate, latency, business metrics for 15 minutes
    → No degradation? Promote to 25%
      → Monitor for 15 minutes
        → No degradation? Promote to 100%
          → v1 pods drained

Any degradation at any step → Automatic rollback to v1
```

### Automated Canary Analysis

Canary analysis compares metrics between the canary (new version) and the baseline (old version) using statistical hypothesis testing:

```yaml
# Canary analysis configuration
analysis:
  interval: 5m
  threshold:
    marginal: 75    # Score below which canary is suspicious
    pass: 95        # Score above which canary passes
  metrics:
    - name: request-error-rate
      provider: prometheus
      query: |
        sum(rate(http_requests_total{status=~"5..",version="{{version}}"}[5m]))
        /
        sum(rate(http_requests_total{version="{{version}}"}[5m]))
      threshold:
        max: 0.01   # Error rate must stay below 1%
    - name: request-latency-p99
      provider: prometheus
      query: |
        histogram_quantile(0.99, rate(http_request_duration_seconds_bucket{version="{{version}}"}[5m]))
      threshold:
        max: 0.5    # P99 must stay below 500ms
```

### Key Metrics to Monitor During Progressive Delivery

| Metric | Alert Threshold | Rollback Trigger |
|--------|----------------|------------------|
| Error rate (5xx) | > 1% above baseline | Immediate rollback |
| P99 latency | > 50% above baseline | Rollback after 2 consecutive intervals |
| Business metric (e.g., conversion rate) | > 5% below baseline | Rollback after 3 consecutive intervals |
| Memory usage | > 20% above baseline | Investigation, potential rollback |

---

## Feature Flag Testing

### The Problem

Feature flags create combinatorial state that is invisible to pre-production testing. A feature that works perfectly when tested alone might break when combined with another flag that changes the data model, API response shape, or UI layout.

### The Solution: Explicit Flag State Testing

CI pipelines explicitly test varying flag states on live systems:

```python
# Test matrix for feature flag combinations
FLAG_STATES = [
    {"new_meeting_ui": True, "realtime_transcription": True},
    {"new_meeting_ui": True, "realtime_transcription": False},
    {"new_meeting_ui": False, "realtime_transcription": True},
    {"new_meeting_ui": False, "realtime_transcription": False},
]

@pytest.mark.parametrize("flags", FLAG_STATES)
def test_meeting_creation_under_flag_combinations(flags):
    """Verify meeting creation works under all flag state combinations."""
    with feature_flags(flags):
        response = create_meeting(title="Test Meeting")
        assert response.status_code == 201
        
        # UI rendering depends on flag state
        if flags["new_meeting_ui"]:
            assert_new_ui_rendered(response)
        else:
            assert_legacy_ui_rendered(response)
```

### Best Practices

- **Test all critical flag combinations.** Not all combinations matter — focus on flags that affect the same user journey or data model.
- **Test flag transitions.** The moment a flag flips from off to on (and back) is when bugs emerge. Test the transition, not just the steady state.
- **Clean up stale flags.** Flags that have been 100% enabled for more than one release cycle should be removed. Dead flags increase test complexity for zero benefit.

---

## Synthetic Monitoring

### What It Is

Automated scripts that continuously execute critical user journeys against production at regular intervals. Unlike real-user monitoring (RUM), synthetic monitors provide consistent, repeatable baselines.

### Implementation

```python
# Synthetic monitor: runs every 5 minutes against production
class MeetingCreationMonitor:
    """
    Validates the full meeting creation journey in production.
    Uses a dedicated test account with isolated data.
    """
    
    def execute(self):
        # Step 1: Authenticate
        token = self.authenticate("synthetic-user@test.internal")
        
        # Step 2: Create meeting
        meeting = self.create_meeting(token, title=f"Synthetic-{uuid4()}")
        assert meeting.status == 201
        
        # Step 3: Verify WebSocket event
        event = self.wait_for_websocket_event("meeting.created", timeout=10)
        assert event is not None
        assert event.meeting_id == meeting.id
        
        # Step 4: Cleanup
        self.delete_meeting(token, meeting.id)
        
        # Report metrics
        self.report_latency("meeting_creation_e2e", elapsed_ms)
```

### Constraints

- **Isolated test accounts.** Synthetic monitors use dedicated accounts that are excluded from analytics and billing.
- **Idempotent operations.** Monitors must clean up after themselves to prevent data accumulation.
- **Alert on failure, not on metrics.** Synthetic monitors should trigger PagerDuty/OpsGenie alerts only after consecutive failures (3+), not on single transient errors.

---

## Observability as Testing

### Trace-Based Production Assertions

The same OpenTelemetry traces that power System Tests provide production validation:

| Pre-Production (System / Live System Test) | Production (Shift-Right) |
|---------------------------|-------------------------|
| Assert span count = 5 | Alert if span count deviates from baseline |
| Assert P99 < 500ms | Alert if P99 exceeds 500ms for 5+ minutes |
| Assert no error status | Alert if error rate exceeds 1% |
| Assert trace completeness | Alert if trace completion rate drops below 99% |

### SLO-Based Quality Gates

Define Service Level Objectives that bridge testing and monitoring:

```
SLO: 99.9% of meeting creation requests complete within 2 seconds
  → System Test: assert P99 < 2000ms in test environment
  → Production: alert if error budget burn rate exceeds threshold
  → Deployment gate: block deploy if pre-production P99 > 1500ms (75% of SLO target)
```

---

## Guardrails

| Guardrail | Rationale |
|-----------|-----------|
| **Never shadow write traffic to shared state** | Shadowed writes can corrupt production databases or trigger duplicate side effects |
| **Automatic rollback must be faster than detection** | If your canary analysis takes 15 minutes, rollback must complete in seconds |
| **Synthetic monitors are not load tests** | Keep synthetic traffic minimal to avoid affecting production metrics |
| **Shift-right does not replace shift-left** | Production testing catches the unknowns. Service Tests and System Tests catch the knowns. Both are required. |

---

## See Also

- `trace-validation.md` — Trace assertions that bridge testing and monitoring
- `pipeline-architecture.md` — Where shift-right sits relative to CI/CD
- `security-testing.md` — WAF and anomaly detection in production
