# Documentation

## Table of Contents
- [Philosophy: Structure Over Comments](#philosophy-structure-over-comments)
- [GoDoc: The Contract Minimum](#godoc-the-contract-minimum)
- [What to Document](#what-to-document)
- [What NOT to Document](#what-not-to-document)
- [Package Documentation](#package-documentation)
- [Concurrency and Safety](#concurrency-and-safety)
- [Deprecation](#deprecation)
- [Inline Comments](#inline-comments)
- [In-Code Markers](#in-code-markers)
- [Service README Template](#service-readme-template)

---

## Philosophy: Structure Over Comments

Comments are promises that no compiler can verify. When code changes, the compiler catches type errors and test failures — but stale comments silently mislead. In a codebase where AI agents drive significant code velocity, comment drift is not hypothetical; it is the default outcome.

**The hierarchy:**
1. **Types** — the compiler rejects incorrect types (zero drift risk)
2. **Naming** — descriptive function/variable names (refactor > comment)
3. **Test names** — executable documentation verified by CI
4. **GoDoc contract comments** — one-sentence summaries for tooling (pkgsite, gopls)
5. **Inline "why" comments** — last resort for genuinely non-obvious decisions

Everything at level 1-3 is verified by tooling. Levels 4-5 are human promises with drift risk. Minimize 4-5 ruthlessly.

---

## GoDoc: The Contract Minimum

Go's `pkgsite` and `gopls` extract doc comments for API reference pages and IDE hover tooltips. This is the only reason exported identifiers need doc comments — they serve as the **contract summary** for consumers who cannot see the implementation.

### The Rule

Every exported identifier gets **one sentence** starting with the identifier name. This sentence describes the contract (what the caller gets), not the implementation (how it works internally).

```go
// Create provisions a new meeting and persists it to the repository.
func (s *MeetingService) Create(ctx context.Context, req domain.CreateMeetingRequest) (*domain.Meeting, error) {

// MeetingRepository defines persistence operations for meetings.
type MeetingRepository interface {

// ErrNotFound is returned when a requested meeting does not exist.
var ErrNotFound = errors.New("meeting not found")

// DefaultTimeout is the request timeout for downstream service calls.
const DefaultTimeout = 30 * time.Second
```

### When to Add More Than One Sentence

Add a second sentence **only** when the information cannot be inferred from the signature:

```go
// Create provisions a new meeting and persists it to the repository.
// It publishes a MeetingCreated event on success.
func (s *MeetingService) Create(ctx context.Context, req domain.CreateMeetingRequest) (*domain.Meeting, error) {
```

The side effect (event publishing) is invisible in the signature — it justifies a second sentence. The persistence detail is already known from the first sentence and the method receiver — do not elaborate further.

---

## What to Document

| What | How | Why |
|------|-----|-----|
| **Exported function/method** | One-sentence GoDoc starting with the name | pkgsite, gopls hover, IDE autocomplete |
| **Exported type** | One-sentence GoDoc describing what it represents | API reference |
| **Exported interface** | One-sentence GoDoc describing the capability it defines | Consumers need to know the contract |
| **Exported errors/constants** | One-sentence GoDoc | Callers need to know when these are returned/used |
| **Side effects** | Add a sentence if the signature does not reveal them | Event publishing, cache invalidation, external calls |
| **Concurrency semantics** | Explicit statement — see [Concurrency and Safety](#concurrency-and-safety) | Cannot be inferred from types |
| **Error conditions** | Document non-obvious return errors | Callers need this for error handling |

---

## What NOT to Document

**Obvious getters:**
```go
// BAD — the signature is self-documenting
// Title returns the title of the meeting.
func (m *Meeting) Title() string { return m.title }

// GOOD — skip the comment entirely
func (m *Meeting) Title() string { return m.title }
```

**Implementation details:**
```go
// BAD — exposes algorithm choice callers don't need to know
// Create uses a B-tree index for O(log n) lookup performance.
func (r *PostgresRepository) Create(ctx context.Context, m *domain.Meeting) error {

// GOOD — documents the contract, not the implementation
// Create persists a new meeting. Returns ErrDuplicate if the ID already exists.
func (r *PostgresRepository) Create(ctx context.Context, m *domain.Meeting) error {
```

**Unexported (lowercase) identifiers** — unless the logic is genuinely non-obvious:
```go
// Skip — unexported, types tell the story
func buildInsertQuery(m *domain.Meeting) (string, []any) {
```

**Generated code** — protobuf, mock generators, etc. have their own documentation.

**Code that should be refactored instead:**
```go
// BAD — comment compensating for unclear code
// Calculate the total duration by summing all segment durations
d := 0
for _, s := range segs {
    d += s.e - s.s
}

// GOOD — naming eliminates the need for a comment
totalDuration := sumSegmentDurations(segments)
```

---

## Package Documentation

Packages get a doc comment only when the package name alone does not convey the purpose.

**Needed** — the package name is generic:
```go
// Package provider implements driven adapters for the meeting service.
// Each provider satisfies a gateway interface defined in internal/core/gateway.
package provider
```

**Not needed** — the package name tells the story:
```go
package meeting // "meeting" is already clear in context
```

For large packages with extensive documentation, use `doc.go`:

```go
// Package provider implements the driven adapters for the meeting service.
//
// # Available Providers
//
//   - PostgresRepository: Persistent storage via pgx/v5
//   - RedisCache: Meeting data caching with TTL
//   - KafkaPublisher: Meeting lifecycle event publishing
package provider
```

---

## Concurrency and Safety

Concurrency semantics are the strongest justification for comments in Go — thread safety **cannot** be inferred from the type signature and is a source of hard-to-debug bugs.

```go
// MeetingCache provides an in-memory cache for meeting data.
// Safe for concurrent use by multiple goroutines.
type MeetingCache struct {

// Compile creates a compiled query plan. Not safe for concurrent use;
// callers must synchronize access or create separate instances.
type QueryCompiler struct {
```

Document the zero-value semantics when it matters:

```go
// Counter tracks event counts. The zero value is ready to use.
type Counter struct {

// Client communicates with the upstream API.
// The zero value is not usable; create instances via NewClient.
type Client struct {
```

---

## Deprecation

Use the `Deprecated:` paragraph to direct users to the replacement:

```go
// GetMeeting retrieves a meeting by its ID.
//
// Deprecated: Use [MeetingService.Get] instead. Will be removed in v3.0.0.
func GetMeeting(ctx context.Context, id string) (*Meeting, error) {
```

---

## Inline Comments

Inline comments within function bodies are justified **only** for genuinely non-obvious "why" that cannot be expressed through naming or structure. If you need a comment to explain *what* code does, refactor the code.

```go
func (s *MeetingService) Create(ctx context.Context, req domain.CreateMeetingRequest) (*domain.Meeting, error) {
    if err := req.Validate(); err != nil {
        return nil, fmt.Errorf("validate meeting: %w", err)
    }

    meeting := domain.NewMeeting(req)

    // Retry with backoff — transient connection errors during pod scaling.
    if err := s.repo.Create(ctx, meeting); err != nil {
        return nil, fmt.Errorf("persist meeting %s: %w", meeting.ID(), err)
    }

    // Notification failures are logged but do not fail meeting creation.
    s.notifier.NotifyAsync(ctx, domain.MeetingCreated{MeetingID: meeting.ID()})

    return meeting, nil
}
```

Notice: the `Validate` and `NewMeeting` calls need no comments — the function names are the documentation.

---

## In-Code Markers

Markers are actionable, not descriptive. They track work, not explain code:

```go
// TODO(alice): Replace with batch insert when pgx v6 adds support. Issue #312.

// FIXME(bob): Race condition with concurrent Create on same idempotency key.
// Needs distributed lock. Issue #247.

// HACK(carol): Workaround for pgx v5 COPY bug — remove after v5.7 upgrade.
```

Always include `(username)` and an issue link for non-trivial items.

---

## Service README Template

```markdown
# [Service Name]

One-sentence description of what this service does.

[![CI](badge-url)](ci-url)
[![Coverage](badge-url)](coverage-url)

## Architecture

\```
internal/
├── core/
│   ├── domain/       # Business entities, value objects, domain errors
│   ├── gateway/      # Outbound port interfaces
│   └── service/      # Application services (orchestration)
├── entrypoints/      # Driving adapters (HTTP, gRPC, CLI)
└── provider/         # Driven adapters (DB, cache, message bus)
\```

## Quick Start

\```bash
git clone https://github.com/org/[service].git && cd [service]
cp .env.example .env
make run
\```

API: `http://localhost:8080` | Health: `GET /health`

## API

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/v1/meetings` | List meetings |
| `POST` | `/v1/meetings` | Create a meeting |
| `GET` | `/v1/meetings/{id}` | Get meeting by ID |

Full spec: [openapi.yaml](docs/openapi.yaml)

## Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `PORT` | `8080` | HTTP server port |
| `DATABASE_URL` | — | PostgreSQL connection string |
| `LOG_LEVEL` | `info` | Logging level |
| `OTEL_EXPORTER_ENDPOINT` | — | OpenTelemetry collector |

## Testing

\```bash
make test          # All tests
make test-unit     # Domain layer (no Docker)
make test-int      # Provider layer (requires Docker)
\```

## License

[MIT](LICENSE)
```

### Tooling

| Tool | Purpose |
|------|---------|
| `pkgsite` | Preview rendered documentation locally |
| `gofmt` | Auto-formats doc comments to canonical syntax |
| `gopls` | IDE hover docs and signature help |
| `golangci-lint` | Enforce doc comment presence via `revive` (`exported` rule) |
