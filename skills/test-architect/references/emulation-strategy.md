# High-Fidelity Emulation Strategy

This module defines how to design, configure, and operate emulator containers that "walk and talk" like production dependencies. The goal is to maintain hermetic Service Test and System Test environments without sacrificing fidelity.

---

## Table of Contents

1. [Core Principle](#core-principle)
2. [Testcontainers Orchestration](#testcontainers-orchestration)
3. [API Mocking with Prism](#api-mocking-with-prism)
4. [Cloud Infrastructure with LocalStack](#cloud-infrastructure-with-localstack)
5. [AI/LLM Emulation with Ollama](#aillm-emulation-with-ollama)
6. [Alternative Emulators](#alternative-emulators)
7. [Container Lifecycle Patterns](#container-lifecycle-patterns)
8. [Networking and Port Management](#networking-and-port-management)
9. [State Management and Determinism](#state-management-and-determinism)

---

## Core Principle

**If a dependency can run in a container, it must not be mocked with an in-memory fake.** In-memory mocks hide critical bugs in:

- SQL query syntax and schema migrations
- Network serialization (protobuf, JSON encoding edge cases)
- Connection pooling and timeout behavior
- Transaction isolation and race conditions
- Data type coercion (timestamps, UUIDs, decimals)

Containerized emulators exercise the real driver, real protocol, and real data format — catching the same class of bugs that production encounters.

---

## Testcontainers Orchestration

Testcontainers is the orchestration layer that manages Docker container lifecycles during test execution. It provides language-native APIs for Go, Python, Java, TypeScript, and more.

### Singleton Container Pattern

Avoid the overhead of starting and stopping containers for every test function. Define containers as shared instances across the test suite.

**Go:**
```go
var postgresContainer testcontainers.Container

func TestMain(m *testing.M) {
    ctx := context.Background()
    req := testcontainers.ContainerRequest{
        Image:        "postgres:16-alpine",
        ExposedPorts: []string{"5432/tcp"},
        Env: map[string]string{
            "POSTGRES_USER":     "test",
            "POSTGRES_PASSWORD": "test",
            "POSTGRES_DB":       "testdb",
        },
        WaitingFor: wait.ForLog("database system is ready to accept connections").
            WithOccurrence(2).
            WithStartupTimeout(30 * time.Second),
    }
    container, err := testcontainers.GenericContainer(ctx, testcontainers.GenericContainerRequest{
        ContainerRequest: req,
        Started:          true,
    })
    if err != nil {
        log.Fatalf("failed to start postgres: %v", err)
    }
    postgresContainer = container

    code := m.Run()

    _ = container.Terminate(ctx)
    os.Exit(code)
}
```

**Python:**
```python
import pytest
from testcontainers.postgres import PostgresContainer

@pytest.fixture(scope="session")
def postgres():
    with PostgresContainer("postgres:16-alpine") as pg:
        yield pg
```

### Smart Wait Strategies

Never assume a container is ready based solely on port availability. Use wait strategies appropriate to the dependency:

| Dependency | Wait Strategy |
|-----------|---------------|
| PostgreSQL | Log message: `"database system is ready to accept connections"` (occurrence: 2) |
| Redis | Log message: `"Ready to accept connections"` |
| Kafka | Log message: `"started (kafka.server.KafkaServer)"` |
| LocalStack | HTTP health check: `GET /_localstack/health` returns 200 |
| Prism | HTTP probe: `GET /` returns any response |
| Ollama | HTTP probe: `GET /api/tags` returns 200 |

### Version Parity

Always pin container image tags to match production versions. Never use `:latest`.

```go
// CORRECT: Pinned to production version
Image: "postgres:16.3-alpine"

// WRONG: Non-deterministic, different behavior per run
Image: "postgres:latest"
```

---

## API Mocking with Prism

**Prism** is an OpenAPI-driven mock server that generates realistic responses based on your API specification. It validates that your requests conform to the spec, catching contract violations at test time.

### Setup via Testcontainers

```go
prismReq := testcontainers.ContainerRequest{
    Image:        "stoplight/prism:5",
    ExposedPorts: []string{"4010/tcp"},
    Cmd:          []string{"mock", "/specs/openapi.yaml", "--host", "0.0.0.0"},
    Mounts: testcontainers.Mounts(
        testcontainers.BindMount("./specs/openapi.yaml", "/specs/openapi.yaml"),
    ),
    WaitingFor: wait.ForHTTP("/").WithPort("4010/tcp"),
}
```

### Controlling Responses

Prism uses the `Prefer` header to control which response example or status code is returned:

```go
// Force a specific status code
req.Header.Set("Prefer", "code=404")

// Force a specific example
req.Header.Set("Prefer", "example=user_not_found")

// Force dynamic generation (random data matching schema)
req.Header.Set("Prefer", "dynamic=true")
```

### Contract Validation

Prism validates incoming requests against the OpenAPI spec. If your service sends a request with a missing required field or wrong type, Prism returns a 422 error — catching contract drift before it reaches production.

---

## Cloud Infrastructure with LocalStack

**LocalStack** emulates AWS services locally, enabling tests that exercise IAM policies, S3 operations, SQS queuing, DynamoDB queries, and Lambda invocations without cloud credentials or billing.

### Setup via Testcontainers

```go
localstackReq := testcontainers.ContainerRequest{
    Image:        "localstack/localstack:3",
    ExposedPorts: []string{"4566/tcp"},
    Env: map[string]string{
        "SERVICES":       "s3,sqs,dynamodb",
        "DEFAULT_REGION": "us-east-1",
    },
    WaitingFor: wait.ForHTTP("/_localstack/health").
        WithPort("4566/tcp").
        WithStatusCodeMatcher(func(status int) bool { return status == 200 }),
}
```

### AWS SDK Configuration

Point your AWS SDK client to the LocalStack endpoint:

```go
cfg, _ := config.LoadDefaultConfig(ctx,
    config.WithRegion("us-east-1"),
    config.WithCredentialsProvider(credentials.NewStaticCredentialsProvider("test", "test", "")),
    config.WithEndpointResolverWithOptions(
        aws.EndpointResolverWithOptionsFunc(func(service, region string, options ...interface{}) (aws.Endpoint, error) {
            return aws.Endpoint{URL: localstackEndpoint}, nil
        }),
    ),
)
```

### Supported Services

LocalStack provides high-fidelity emulation for: S3, SQS, SNS, DynamoDB, Lambda, API Gateway, CloudFormation, IAM, KMS, Secrets Manager, Step Functions, and more. Choose the services your application requires via the `SERVICES` environment variable.

---

## AI/LLM Emulation with Ollama

**Ollama** runs local language models that implement the OpenAI-compatible API specification. Use it for Service Tests and System Tests where LLM behavior must be validated without incurring API costs.

### Setup via Testcontainers

```go
ollamaReq := testcontainers.ContainerRequest{
    Image:        "ollama/ollama:latest",
    ExposedPorts: []string{"11434/tcp"},
    WaitingFor:   wait.ForHTTP("/api/tags").WithPort("11434/tcp"),
}
```

### Model Selection for Testing

Use small, fast models that are sufficient for validating integration logic without requiring production-grade output quality:

| Purpose | Recommended Model | Size |
|---------|------------------|------|
| API integration validation | `tinyllama` | ~600MB |
| Structured output parsing | `phi3:mini` | ~2.3GB |
| Prompt template testing | `llama3.2:1b` | ~1.3GB |

### When Ollama is Insufficient

Promote to Live Service Test when:

- Testing prompt engineering quality (nuanced output evaluation)
- Validating model-specific behavior (token limits, safety filters)
- Benchmarking response latency against production SLAs

---

## Alternative Emulators

| Tool | Use Case | When to Prefer |
|------|----------|----------------|
| **WireMock** | Complex HTTP mocking with stateful behavior, request matching, fault injection | Need to simulate specific failure modes (timeouts, connection resets) or complex request/response sequences |
| **Mountebank** | Multi-protocol mocking (HTTP, TCP, SMTP) | Mocking non-HTTP protocols or need cross-protocol test scenarios |
| **LocalAI** | OpenAI-compatible API with broader model support | Need specific model architectures not available in Ollama |

---

## Container Lifecycle Patterns

### Per-Suite (Recommended Default)

Start containers once for the entire test suite. Reset state between tests via truncation or transactional rollback.

```
Suite Start → Start Containers → Run All Tests → Terminate Containers
```

**Pros:** Fast. Minimal container startup overhead.
**Cons:** Requires careful state isolation between tests.

### Per-Test (High Isolation)

Start fresh containers for each test function. Use when tests modify container configuration or when state cleanup is impractical.

```
Test Start → Start Containers → Run Test → Terminate Containers → Next Test
```

**Pros:** Perfect isolation.
**Cons:** Slow. Only use when state reset is impossible.

### State Reset Between Tests

For per-suite lifecycle, reset database state between tests:

```go
func resetDatabase(t *testing.T, db *sql.DB) {
    t.Helper()
    tables := []string{"meetings", "tasks", "users"}
    for _, table := range tables {
        _, err := db.Exec(fmt.Sprintf("TRUNCATE TABLE %s CASCADE", table))
        require.NoError(t, err)
    }
}
```

---

## Networking and Port Management

### Dynamic Port Allocation

**Never use fixed host ports.** Testcontainers dynamically maps container ports to random available host ports, preventing collisions in parallel CI execution.

```go
// CORRECT: Dynamic port
mappedPort, _ := container.MappedPort(ctx, "5432/tcp")
dsn := fmt.Sprintf("postgres://test:test@localhost:%s/testdb", mappedPort.Port())

// WRONG: Fixed port — breaks parallel execution
ExposedPorts: []string{"5432:5432"}
```

### Shared Docker Networks

When multiple containers need to communicate (e.g., service container calling a Prism mock), create a shared Docker network:

```go
network, _ := testcontainers.GenericNetwork(ctx, testcontainers.GenericNetworkRequest{
    NetworkRequest: testcontainers.NetworkRequest{Name: "test-network"},
})

// Both containers join the same network
containerReq.Networks = []string{"test-network"}
```

Within a shared network, containers can reference each other by container name rather than dynamic host ports, closely mimicking production service discovery.

---

## State Management and Determinism

### Principles

1. **Recreate or reset per suite.** Container state from a previous test suite must never leak into the current run.
2. **Seed deterministically.** If tests require pre-existing data, use migration scripts or fixture SQL that produce identical state every time.
3. **Isolate writes.** If tests run in parallel against a shared container, use unique identifiers (UUIDs, test-scoped namespaces) to prevent cross-test interference.

### Database Seeding Pattern

```go
func seedTestData(t *testing.T, db *sql.DB) {
    t.Helper()
    _, err := db.Exec(`
        INSERT INTO users (id, email, name) VALUES
            ('usr_001', 'alice@test.com', 'Alice'),
            ('usr_002', 'bob@test.com', 'Bob')
        ON CONFLICT DO NOTHING
    `)
    require.NoError(t, err)
}
```

---

## See Also

- `validation-layers.md` — When to use emulated vs live dependencies
- `test-data-management.md` — Container reset strategies and fixture patterns
- `trace-validation.md` — Configuring OTel collectors within emulated environments
