# Test Taxonomy & Naming Standards

This module defines the universal classification system and naming conventions for all automated tests. Consistent taxonomy enables automated agents to parse, route, quarantine, and analyze test results across millions of executions.

---

## Table of Contents

1. [The S/M/L Size Taxonomy](#the-sml-size-taxonomy)
2. [Fully Qualified Names (FQN)](#fully-qualified-names-fqn)
3. [BDD Naming Convention](#bdd-naming-convention)
4. [Metadata Standards](#metadata-standards)
5. [Practical Examples](#practical-examples)
6. [Antipatterns](#antipatterns)

---

## The S/M/L Size Taxonomy

Tests are classified by **environmental constraints**, not by functional type. This ensures CI/CD pipelines can partition test execution by resource cost and parallelism potential.

### Small (Unit)

- **Process Boundary:** Single process only. No forking, no subprocesses.
- **I/O:** No blocking I/O. No filesystem, no network, no database.
- **Completion Target:** < 100ms per test.
- **Parallelism:** Unlimited — these tests have zero shared state.
- **When to Use:** Pure algorithmic logic, data transformations, math functions, validators, serializers. These are the exceptions, not the default.

### Medium (Integration / Service Test)

- **Process Boundary:** Multiple processes allowed. Local network I/O.
- **I/O:** Local databases (Testcontainers), local network calls, filesystem access.
- **Completion Target:** < 5 seconds per test.
- **Parallelism:** Constrained by container resources. Use dynamic port allocation.
- **When to Use:** This is the primary validation layer. Test from the API entry point through business logic to real database containers and emulated third-party APIs.

### Large (System / E2E)

- **Process Boundary:** Full system deployment. Multiple services, real network calls.
- **I/O:** External dependencies (emulated or live). Full network stack.
- **Completion Target:** Varies. Budget generously.
- **Parallelism:** Limited — typically sequential or lightly parallelized.
- **When to Use:** Multi-service integration validation, end-to-end user journeys, trace-driven validation, chaos experiments.

### CI/CD Pipeline Mapping

```
┌─────────────────────────────────────────────────┐
│ PR Pipeline (Every Commit)                      │
│  ├── Small tests (~seconds)                     │
│  ├── Medium tests (~minutes)                    │
│  └── Static analysis (SAST, linting, types)     │
├─────────────────────────────────────────────────┤
│ Merge Pipeline (Trunk / Main)                   │
│  ├── All Small + Medium tests                   │
│  ├── Large tests (emulated 3rd parties)         │
│  └── Contract verification (CDCT)               │
├─────────────────────────────────────────────────┤
│ Manual Trigger Only (Developer-Initiated)       │
│  ├── Live Service Tests                        │
│  └── Live System Tests                         │
├─────────────────────────────────────────────────┤
│ Scheduled Pipeline (Nightly)                    │
│  ├── Mutation testing                           │
│  └── Chaos experiments                          │
└─────────────────────────────────────────────────┘
```

---

## Fully Qualified Names (FQN)

Every test must be addressable via a unique, hierarchical string that enables automated analysis, ownership routing, and cone queries.

### Format

```
[realm].[path].[target].[test_name]
```

### Components

| Component | Description | Example |
|-----------|-------------|---------|
| `realm` | Language or runtime ecosystem | `golang`, `python`, `typescript` |
| `path` | Test type and structural path | `service_test.infrastructure.database` |
| `target` | The specific module or component | `user_repository` |
| `test_name` | BDD-style descriptive name | `find_by_id_should_return_user_when_exists` |

### Full Examples

```
golang.service_test.infrastructure.database.user_repository.find_by_id_should_return_user_when_exists
python.service_test.domain.transcription.audio_codec.convert_should_produce_wav_when_given_mp3
typescript.system_test.e2e.meeting_flow.create_meeting_should_emit_websocket_event_when_user_authenticated
```

### Cone Queries

FQN enables hierarchical filtering for analysis:

```
# All Go service tests
golang.service_test.*

# All database-related tests across all languages
*.service_test.infrastructure.database.*

# All tests for a specific module
*.*.*.user_repository.*
```

**Benefit:** Automated agents use cone queries to track flakiness rates, auto-ticket the owning team via ownership mappings, and identify performance regressions across millions of test executions.

---

## BDD Naming Convention

Every test name follows the Behavior-Driven Development pattern, ensuring that failure logs provide immediate diagnostic value.

### Format

```
[Function] should [Expected Outcome] when [Condition]
```

### Rules

1. **Use natural language.** Avoid abbreviations, single-letter variables, or cryptic identifiers.
2. **State the behavior, not the implementation.** The test name should survive a complete refactor of internal logic.
3. **Include the boundary condition.** The `when` clause captures the specific scenario that makes this test unique.

### Examples

**Go:**
```go
func TestCreateMeeting_ShouldReturnConflict_WhenMeetingAlreadyExists(t *testing.T) { ... }
func TestTranscribe_ShouldEmitProgressEvent_WhenAudioExceedsFiveMinutes(t *testing.T) { ... }
```

**Python:**
```python
def test_create_task_should_return_422_when_source_enum_is_invalid():
    ...

def test_convert_audio_should_produce_wav_when_given_mp3_input():
    ...
```

**TypeScript:**
```typescript
it('should render meeting list when user has active meetings', () => { ... });
it('should show empty state when no meetings exist', () => { ... });
```

### Antipattern: What NOT to Do

```go
// BAD: Cryptic, no behavior, no condition
func TestCreate(t *testing.T) { ... }
func TestMeeting1(t *testing.T) { ... }
func Test_handler_post(t *testing.T) { ... }
```

---

## Metadata Standards

Consistent metadata formats ensure machine readability across all tooling, dashboards, and automated analysis pipelines.

### Timestamps

- **Format:** ISO 8601 strictly. Example: `2026-04-16T12:00:00Z`.
- **Timezone:** Always UTC for storage. Convert to local only at display time.

### Delimiters

- **Metadata Fields:** Use underscores `_` for all metadata field names.
- **Rationale:** Ensures compatibility with data ingestion tools, log aggregators, and query engines that may interpret hyphens as operators.

### Test Result Schema

```json
{
  "test_fqn": "golang.service_test.infrastructure.database.user_repository.find_by_id_should_return_user_when_exists",
  "test_size": "medium",
  "test_type": "service_test",
  "duration_ms": 342,
  "status": "passed",
  "timestamp": "2026-04-16T12:00:00Z",
  "trace_id": "abc123def456",
  "flakiness_score": 0.02,
  "owning_team": "core-backend"
}
```

---

## Practical Examples

### Mapping a Feature to the Taxonomy

**Feature:** User creates a meeting via REST API, which persists to PostgreSQL and emits a WebSocket event.

| Test | Size | Type | FQN |
|------|------|------|-----|
| Meeting validator rejects empty title | Small | Unit | `golang.unit.domain.meeting.validate_should_reject_when_title_empty` |
| Create meeting persists and returns 201 | Medium | Service Test | `golang.service_test.api.meeting.create_should_return_201_when_valid_payload` |
| Create meeting emits WebSocket event | Medium | Service Test | `golang.service_test.api.meeting.create_should_emit_ws_event_when_persisted` |
| Meeting creation visible in frontend | Large | System Test | `typescript.system_test.e2e.meeting.create_should_render_in_list_when_created_via_api` |

---

## Antipatterns

| Antipattern | Problem | Correct Approach |
|-------------|---------|------------------|
| `test_1`, `test_2`, `test_3` | Zero diagnostic value on failure | Use BDD naming with behavior and condition |
| No size classification | CI cannot partition or parallelize | Assign S/M/L based on environmental boundaries |
| Inconsistent FQN formats | Automated analysis and cone queries fail | Enforce `realm.path.target.test_name` universally |
| Timestamps in local time | Data aggregation across timezones breaks | Store as ISO 8601 UTC |
| Hyphens in metadata fields | Some ingestion tools interpret as minus operator | Use underscores for all metadata fields |

---

## See Also

- `validation-layers.md` — How test types map to the Validation Grid
- `worked-examples.md` — End-to-end examples showing taxonomy applied to real features
- `pipeline-architecture.md` — How S/M/L sizes map to pipeline stages
