# Consumer-Driven Contract Testing (CDCT)

This module defines how to implement Consumer-Driven Contract Testing to decouple service teams, eliminate shared staging dependencies, and enable independent deployment velocity.

---

## Table of Contents

1. [Core Principle](#core-principle)
2. [The Integrated Test Problem](#the-integrated-test-problem)
3. [CDCT Architecture](#cdct-architecture)
4. [Consumer Side: Defining Expectations](#consumer-side-defining-expectations)
5. [Provider Side: Verification](#provider-side-verification)
6. [Pact Broker Integration](#pact-broker-integration)
7. [CI/CD Pipeline Integration](#cicd-pipeline-integration)
8. [AsyncAPI / Event-Driven Contracts](#asyncapi--event-driven-contracts)
9. [Best Practices](#best-practices)

---

## Core Principle

**Consumers define their expectations. Providers verify them before deployment.**

This inverts the traditional testing model. Instead of consumers writing integration tests against a running provider (which creates fragile coupling), consumers declare what they need, and providers independently verify they can satisfy those declarations.

The result: teams deploy independently, staging environments become unnecessary for inter-service validation, and contract violations are caught at build time — not in production.

---

## The Integrated Test Problem

"Integrated Tests" — tests that depend on the live state of another team's service — are a primary source of pipeline instability:

| Problem | Impact |
|---------|--------|
| Provider is down for maintenance | Consumer's CI pipeline fails for reasons unrelated to their code |
| Provider deploys a breaking change | Consumer discovers it only after their own deployment fails |
| Shared staging environment drift | "Works on my machine" but fails in staging due to stale configuration |
| "Stepping on toes" | Multiple teams modifying shared test data in the same staging environment |

CDCT eliminates all of these by removing the runtime dependency between consumer and provider during testing.

---

## CDCT Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Pact Broker                              │
│  (Central registry of all consumer expectations)                │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │ Contract A↔B │  │ Contract A↔C │  │ Contract B↔C │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────────┐          ┌──────────────────┐            │
│  │ Consumer (App)   │          │ Provider (Core)  │            │
│  │                  │          │                  │            │
│  │ 1. Writes Pact   │  publish │ 3. Downloads     │            │
│  │    test defining  │ ──────► │    consumer Pact  │            │
│  │    expectations   │         │                  │            │
│  │                  │          │ 4. Replays Pact   │            │
│  │ 2. Publishes     │         │    against real   │            │
│  │    contract to    │         │    provider API   │            │
│  │    broker        │          │                  │            │
│  │                  │          │ 5. Results back   │            │
│  │                  │  verify  │    to broker      │            │
│  │                  │ ◄─────── │                  │            │
│  └──────────────────┘          └──────────────────┘            │
└─────────────────────────────────────────────────────────────────┘
```

### The Flow

1. **Consumer** writes a Pact test that defines the HTTP requests it makes and the responses it expects.
2. **Consumer CI** publishes the generated Pact contract to the Pact Broker.
3. **Provider CI** downloads all consumer Pact contracts from the broker.
4. **Provider** replays each consumer's requests against its real API and verifies responses match expectations.
5. **Provider** publishes verification results back to the broker.
6. **Deployment Gate:** The Pact Broker's `can-i-deploy` tool checks if all contracts are verified before allowing deployment.

---

## Consumer Side: Defining Expectations

The consumer writes tests that define exactly which endpoints it calls and what response shape it needs.

### Go Consumer Example

```go
func TestMeetingServiceConsumer(t *testing.T) {
    // Configure the Pact mock server
    pact := dsl.Pact{
        Consumer: "wordloop-app",
        Provider: "wordloop-core",
    }
    defer pact.Teardown()

    // Define the interaction
    pact.AddInteraction().
        Given("a meeting exists with ID meeting_001").
        UponReceiving("a request to get meeting details").
        WithRequest(dsl.Request{
            Method: "GET",
            Path:   dsl.String("/api/v1/meetings/meeting_001"),
            Headers: dsl.MapMatcher{
                "Authorization": dsl.String("Bearer valid-token"),
            },
        }).
        WillRespondWith(dsl.Response{
            Status: 200,
            Headers: dsl.MapMatcher{
                "Content-Type": dsl.String("application/json"),
            },
            Body: dsl.Match(MeetingResponse{
                ID:     "meeting_001",
                Title:  dsl.Like("Team Standup"),
                Status: dsl.Term("active", "active|completed|cancelled"),
            }),
        })

    // Execute: call the consumer's client against the Pact mock server
    err := pact.Verify(func() error {
        client := NewMeetingClient(pact.Server.Port)
        meeting, err := client.GetMeeting("meeting_001")
        assert.Equal(t, "meeting_001", meeting.ID)
        return err
    })
    assert.NoError(t, err)
}
```

### Python Consumer Example

```python
import pytest
from pact import Consumer, Provider

@pytest.fixture
def pact():
    pact = Consumer("wordloop-ml").has_pact_with(
        Provider("wordloop-core"),
        pact_dir="./pacts"
    )
    pact.start_service()
    yield pact
    pact.stop_service()

def test_create_task_contract(pact):
    """Consumer defines what it expects when creating a transcription task."""
    expected_body = {
        "id": "task_001",
        "status": "pending",
        "source": "audio_upload"
    }
    
    (pact
        .given("authenticated as service account")
        .upon_receiving("a request to create a transcription task")
        .with_request("POST", "/api/v1/tasks", body={
            "meeting_id": "meeting_001",
            "source": "audio_upload"
        })
        .will_respond_with(201, body=expected_body))
    
    with pact:
        client = TaskClient(base_url=pact.uri)
        task = client.create_task(meeting_id="meeting_001", source="audio_upload")
        assert task.id == "task_001"
        assert task.status == "pending"
```

---

## Provider Side: Verification

The provider downloads consumer contracts and replays them against its real API. No mocks — the provider runs its full application stack (with Testcontainers for databases).

### Go Provider Verification

```go
func TestPactProvider(t *testing.T) {
    // Start the real provider with Testcontainers
    app := startTestServer(t) // uses real DB via Testcontainers
    
    pact := dsl.Pact{
        Provider: "wordloop-core",
    }
    
    // Define state handlers for "Given" clauses
    stateHandlers := dsl.StateHandlers{
        "a meeting exists with ID meeting_001": func() error {
            return seedMeeting(app.DB, "meeting_001", "Team Standup")
        },
        "authenticated as service account": func() error {
            return seedServiceToken(app.DB, "valid-service-token")
        },
    }
    
    // Verify all consumer contracts
    _, err := pact.VerifyProvider(t, dsl.VerifyRequest{
        ProviderBaseURL:            app.BaseURL,
        PactBrokerURL:             "https://pact-broker.internal",
        ProviderVersion:            os.Getenv("GIT_SHA"),
        PublishVerificationResults: true,
        StateHandlers:              stateHandlers,
        ConsumerVersionSelectors: []dsl.ConsumerVersionSelector{
            {Tag: "main", Latest: true},
        },
    })
    assert.NoError(t, err)
}
```

---

## Pact Broker Integration

The Pact Broker is the central registry that coordinates contract publishing, verification, and deployment gating.

### Key Capabilities

| Feature | Purpose |
|---------|---------|
| **Contract Storage** | Stores all consumer Pact files, versioned by consumer version and branch |
| **Verification Results** | Tracks which provider versions have verified which consumer contracts |
| **can-i-deploy** | CLI tool that checks whether a version is safe to deploy based on verification status |
| **Webhooks** | Triggers provider CI when a new consumer contract is published |
| **Network Diagram** | Visualizes service dependencies based on contract relationships |

### Deployment Gate: can-i-deploy

```bash
# Check if wordloop-core v1.2.3 can be deployed
pact-broker can-i-deploy \
  --pacticipant wordloop-core \
  --version $(git rev-parse HEAD) \
  --to-environment production

# Output:
# COMPUTER SAYS YES
# All required verification results are published and successful
```

---

## CI/CD Pipeline Integration

### Consumer Pipeline

```yaml
# .github/workflows/consumer-ci.yml
steps:
  - name: Run Pact Tests
    run: go test ./contracts/... -v
    
  - name: Publish Pacts to Broker
    run: |
      pact-broker publish ./pacts \
        --consumer-app-version $(git rev-parse HEAD) \
        --branch $(git branch --show-current) \
        --broker-base-url https://pact-broker.internal
```

### Provider Pipeline

```yaml
# .github/workflows/provider-ci.yml
steps:
  - name: Verify Consumer Contracts
    run: go test ./contracts/provider/... -v
    env:
      PACT_BROKER_URL: https://pact-broker.internal
      GIT_SHA: ${{ github.sha }}
      
  - name: Can I Deploy?
    run: |
      pact-broker can-i-deploy \
        --pacticipant wordloop-core \
        --version ${{ github.sha }} \
        --to-environment production
```

### Webhook-Triggered Verification

When a consumer publishes a new contract, the Pact Broker triggers a webhook to the provider's CI:

```
Consumer publishes new Pact
  → Broker webhook fires
    → Provider CI runs verification job
      → Results published back to broker
        → Consumer's `can-i-deploy` gate passes
```

---

## AsyncAPI / Event-Driven Contracts

CDCT extends beyond HTTP to event-driven systems. For Pub/Sub, WebSocket, and message-based communication, contract testing validates message schemas.

### Message Pact Example

```go
func TestWebSocketEventContract(t *testing.T) {
    pact := dsl.Pact{
        Consumer: "wordloop-app",
        Provider: "wordloop-core",
    }
    
    pact.AddMessageInteraction().
        Given("a meeting was just created").
        ExpectsToReceive("a meeting.created CloudEvent").
        WithMetadata(dsl.MapMatcher{
            "type":   dsl.String("com.wordloop.meeting.created"),
            "source": dsl.String("/services/wordloop-core"),
        }).
        WithContent(dsl.Match(MeetingCreatedPayload{
            MeetingID: dsl.Like("meeting_001"),
            Title:     dsl.Like("Team Standup"),
            CreatedBy: dsl.Like("user_001"),
        }))
}
```

---

## Best Practices

| Practice | Rationale |
|----------|-----------|
| **Test only what you consume** | Consumer Pacts should only define the fields and endpoints the consumer actually uses, not the provider's full API surface. |
| **Use matchers, not exact values** | `dsl.Like()` and `dsl.Term()` allow flexible matching — the exact value does not matter, only the type and format. |
| **Version by Git SHA** | Both consumer and provider versions should be tied to Git commits for full traceability. |
| **State handlers are test fixtures** | Provider state handlers (`Given` clauses) seed the real database with test data. They are Service Test fixtures. |
| **Run `can-i-deploy` before every deployment** | This is the deployment safety net. It prevents deploying a provider that would break a consumer. |
| **Publish verification results** | Without published results, `can-i-deploy` cannot make informed decisions. |

---

## See Also

- `validation-layers.md` — Where contract tests sit in the Validation Grid
- `test-data-management.md` — State handlers as factory fixtures
- `pipeline-architecture.md` — CDCT verification in the Merge pipeline
