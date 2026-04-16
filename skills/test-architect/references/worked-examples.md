# Worked Examples

This module provides end-to-end decision walkthroughs that demonstrate how the test architect applies the Validation Grid, emulation strategy, risk scoring, naming conventions, and trace assertions together for a concrete feature.

---

## Table of Contents

1. [Example 1: REST Endpoint with Database and WebSocket](#example-1-rest-endpoint-with-database-and-websocket)
2. [Example 2: ML Pipeline with External LLM](#example-2-ml-pipeline-with-external-llm)
3. [Example 3: Event-Driven Cross-Service Feature](#example-3-event-driven-cross-service-feature)

---

## Example 1: REST Endpoint with Database and WebSocket

### Feature Description

> **POST /api/v1/meetings** — Creates a new meeting. Persists to PostgreSQL, emits a `meeting.created` CloudEvent via WebSocket, and returns the created meeting with a 201 status.

### Step 1: Classify the Scope (Validation Grid)

**Questions:**
- How many services are involved? → **One** (wordloop-core handles creation, persistence, and WebSocket emission)
- Are any 3rd-party APIs called? → **No** (PostgreSQL and WebSocket are owned infrastructure)

**Decision:** This is a **Service Test** (single service, emulated dependencies).

**Also consider:** A **System Test** for the cross-service journey where the frontend receives the WebSocket event and renders the new meeting.

### Step 2: Design the Emulation Strategy

| Dependency | Emulation | Rationale |
|-----------|-----------|-----------|
| PostgreSQL | Testcontainers (`postgres:16-alpine`) | Real SQL execution, migration validation |
| Redis (WebSocket pubsub) | Testcontainers (`redis:7-alpine`) | Real pubsub behavior |
| WebSocket | Real (in-process) | Part of the service under test |

No 3rd-party APIs → no Prism/LocalStack/Ollama needed.

### Step 3: Score Risk

| Parameter | Score | Rationale |
|-----------|-------|-----------|
| Impact | 4 | Core user-facing feature. Data integrity critical. |
| Complexity | 3 | Database write + WebSocket emit + validation |
| Historical Failure Rate | 2 | Stable area, occasional serialization bugs |

**Risk Score:** 4 × 3 × 2 = **24** → **High**

**Recommended depth:** Service Tests + System Tests + mutation testing on critical assertions.

### Step 4: Define Test Structure

```
golang.service_test.api.meeting.create_should_return_201_when_valid_payload
golang.service_test.api.meeting.create_should_return_400_when_title_empty
golang.service_test.api.meeting.create_should_return_400_when_title_exceeds_max_length
golang.service_test.api.meeting.create_should_return_401_when_no_auth_token
golang.service_test.api.meeting.create_should_return_409_when_duplicate_title_for_user
golang.service_test.api.meeting.create_should_persist_to_database_when_valid_payload
golang.service_test.api.meeting.create_should_emit_websocket_event_when_persisted
golang.service_test.api.meeting.create_should_return_403_when_user_exceeds_meeting_limit

typescript.system_test.e2e.meeting.create_should_render_in_list_when_created_via_api
typescript.system_test.e2e.meeting.create_should_receive_ws_event_when_another_user_creates
```

### Step 5: Trace Assertions (System Test only)

```
Expected trace for POST /api/v1/meetings:
  
  MeetingHandler.Create (root span)
    ├── MeetingService.Create
    │     ├── MeetingRepository.Save (db.system=postgresql)
    │     └── WebSocketHub.Broadcast (messaging.destination=meetings)
    └── Response: 201 Created

Assertions:
  - Span count = 4
  - MeetingRepository.Save is child of MeetingService.Create
  - WebSocketHub.Broadcast has attribute messaging.destination = "meetings"
  - No spans have status = ERROR
  - Root span duration < 500ms
```

### Step 6: Contract Boundaries

**Consumer:** `wordloop-app` (frontend) consumes `GET /api/v1/meetings/:id` and `meeting.created` WebSocket events.

**Pact contract defines:**
- Response shape for `GET /api/v1/meetings/:id` (id, title, status, created_at)
- CloudEvent schema for `meeting.created` (meeting_id, title, created_by)

### Step 7: Implementation Skeleton

```go
func TestCreateMeeting_ShouldReturn201_WhenValidPayload(t *testing.T) {
    // Arrange
    user := NewUserBuilder().Build(t, db)
    token := createToken(t, user.ID)
    
    // Act
    resp := httpPost(t, "/api/v1/meetings",
        withJSON(map[string]any{"title": "Team Standup", "duration": 30}),
        withAuth(token))
    
    // Assert: HTTP response
    assert.Equal(t, 201, resp.StatusCode)
    
    var body map[string]any
    json.NewDecoder(resp.Body).Decode(&body)
    assert.NotEmpty(t, body["id"])
    assert.Equal(t, "Team Standup", body["title"])
    
    // Assert: Database persistence
    var count int
    db.QueryRow("SELECT COUNT(*) FROM meetings WHERE id = $1", body["id"]).Scan(&count)
    assert.Equal(t, 1, count)
    
    // Assert: WebSocket event emitted
    event := wsClient.WaitForEvent(t, "meeting.created", 5*time.Second)
    assert.Equal(t, body["id"], event.Data["meeting_id"])
}
```

---

## Example 2: ML Pipeline with External LLM

### Feature Description

> **POST /api/v1/tasks** — Creates a transcription task. Accepts an audio file reference, calls OpenAI Whisper for transcription, stores the result, and emits a `task.completed` event via Pub/Sub.

### Step 1: Classify the Scope

**Questions:**
- How many services? → **One** (wordloop-ml)
- 3rd-party APIs? → **Yes** (OpenAI Whisper)

**Decision:**
- **Service Test** with Ollama emulating OpenAI (primary, automated)
- **Live Service Test** for prompt quality validation (manual, during integration development)

### Step 2: Emulation Strategy

| Dependency | Service Test | Live Service Test |
|-----------|-------------|-------------------|
| PostgreSQL | Testcontainers | Testcontainers |
| OpenAI Whisper | **Ollama** (local model) | **Real OpenAI API** |
| S3 (audio storage) | **LocalStack** | **LocalStack** |
| Pub/Sub | Testcontainers (Redis) | Testcontainers (Redis) |

### Step 3: Score Risk

| Parameter | Score | Rationale |
|-----------|-------|-----------|
| Impact | 5 | Core product feature, user-facing transcription quality |
| Complexity | 5 | Audio processing + LLM inference + async pipeline |
| Historical Failure Rate | 4 | Frequent regressions due to model behavior changes |

**Risk Score:** Capped at **25** → **High**

**Recommended depth:** Full Service Test + Live Service Test suite + mutation testing.

### Step 4: Test Structure

```
python.service_test.domain.transcription.create_task_should_return_201_when_valid_audio
python.service_test.domain.transcription.create_task_should_return_422_when_invalid_format
python.service_test.domain.transcription.transcribe_should_produce_text_when_given_wav
python.service_test.domain.transcription.transcribe_should_emit_completed_event_when_done
python.service_test.domain.transcription.transcribe_should_store_result_in_s3_when_completed

python.live_service_test.domain.transcription.transcribe_should_capture_key_topics_when_real_audio
python.live_service_test.domain.transcription.transcribe_should_handle_rate_limit_when_concurrent
```

### Step 5: Key Decision — When to Promote to Live

```
Use Ollama (Service Test) when:
  ✓ Testing the pipeline mechanics (audio in → text out → event emitted)
  ✓ Validating error handling (invalid audio, timeout, API failure)
  ✓ Running in CI on every PR

Promote to Live Service Test when:
  ✓ Evaluating transcription quality (accuracy, topic extraction)
  ✓ Testing prompt engineering changes
  ✓ Validating rate limit handling against real API
  ✓ Developer is actively iterating on the OpenAI integration
```

---

## Example 3: Event-Driven Cross-Service Feature

### Feature Description

> When a meeting is created in `wordloop-core`, an event triggers `wordloop-ml` to prepare a transcription workspace. The frontend (`wordloop-app`) receives a WebSocket notification that the workspace is ready.

### Step 1: Classify the Scope

**Questions:**
- How many services? → **Three** (core, ml, app)
- 3rd-party APIs? → **No** (all internal)

**Decision:** This is a **System Test** (multi-service, emulated).

### Step 2: Emulation Strategy

```
┌──────────────────────────────────────────────────────┐
│ Test Runner (go test)                                │
│  → Creates meeting via Core API                      │
│  → Asserts ML workspace created                      │
│  → Asserts WebSocket event received by App           │
│  → Validates OTel trace end-to-end                   │
├──────────────────────────────────────────────────────┤
│ Service Cluster (Docker Compose)                     │
│  ├── wordloop-core (Go)                              │
│  ├── wordloop-ml (Python/FastAPI)                    │
│  └── wordloop-app (Next.js)                          │
├──────────────────────────────────────────────────────┤
│ Infrastructure                                       │
│  ├── PostgreSQL (shared, per-service schemas)        │
│  ├── Redis (WebSocket + Pub/Sub)                     │
│  ├── Ollama (LLM emulation)                          │
│  ├── LocalStack (S3)                                 │
│  └── OTel Collector → Jaeger                         │
└──────────────────────────────────────────────────────┘
```

### Step 3: Trace Assertion

```
Expected trace for meeting creation → workspace preparation:

  MeetingHandler.Create (wordloop-core)
    ├── MeetingService.Create
    │     ├── MeetingRepository.Save
    │     └── PubSub.Publish("meeting.created")
    │
    └── [async, triggered by event]
        TaskHandler.OnMeetingCreated (wordloop-ml)
          ├── WorkspaceService.Prepare
          │     ├── S3Client.CreateBucket
          │     └── TranscriptionConfig.Initialize
          └── PubSub.Publish("workspace.ready")
              │
              └── [async, triggered by event]
                  WebSocketHub.Broadcast (wordloop-core)
                    └── Client receives "workspace.ready"

Assertions:
  - Single TraceID spans ALL three services
  - wordloop-ml spans are children of wordloop-core's PubSub.Publish
  - WebSocket broadcast is child of wordloop-ml's PubSub.Publish
  - Total trace duration < 5000ms
  - No spans have status = ERROR
```

### Step 4: Test Implementation

```go
func TestMeetingCreation_ShouldPrepareWorkspace_WhenMeetingCreated(t *testing.T) {
    // Arrange
    user := seedUserViaAPI(t, coreClient)
    wsConn := connectWebSocket(t, user.Token)
    
    // Act: Create meeting via Core API
    meeting := createMeetingViaAPI(t, coreClient, user.Token, "Team Standup")
    
    // Assert: Workspace prepared in ML service
    require.Eventually(t, func() bool {
        workspace, err := mlClient.GetWorkspace(meeting.ID)
        return err == nil && workspace.Status == "ready"
    }, 10*time.Second, 500*time.Millisecond)
    
    // Assert: WebSocket event received
    event := wsConn.WaitForEvent(t, "workspace.ready", 10*time.Second)
    assert.Equal(t, meeting.ID, event.Data["meeting_id"])
    
    // Assert: Trace completeness
    traceID := meeting.TraceID
    assertTraceComplete(t, traceID, TraceExpectation{
        MinSpanCount: 8,
        RequiredSpans: []string{
            "MeetingService.Create",
            "PubSub.Publish",
            "WorkspaceService.Prepare",
            "WebSocketHub.Broadcast",
        },
        MaxDuration: 5 * time.Second,
    })
}
```

---

## Pattern: Applying the Test Architect Workflow

Every example above follows the same sequence:

1. **Classify** → Validation Grid (scope × fidelity)
2. **Emulate** → Select containers per dependency
3. **Score** → Risk Matrix (impact × complexity × failure rate)
4. **Name** → FQN + BDD naming
5. **Trace** → Span expectations for System Tests
6. **Contract** → CDCT boundaries (if multi-consumer)
7. **Implement** → Actual test code with factories, assertions, and cleanup

---

## See Also

- `validation-layers.md` — The Validation Grid applied in each example
- `test-data-management.md` — Factory patterns used in test implementations
- `trace-validation.md` — Trace assertion patterns demonstrated in examples
