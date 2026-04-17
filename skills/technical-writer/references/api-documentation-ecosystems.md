# API Documentation Ecosystems

API documentation is not a static reference page. It is an active, interactive environment engineered to reduce **time-to-first-call** from hours to minutes. The API specification is the single source of truth — it generates the documentation, the SDKs, the playgrounds, and the contract tests. Everything flows from the spec, never the reverse.

---

## Table of Contents
1. [Single Source of Truth](#single-source-of-truth)
2. [OpenAPI Specification Standards](#openapi-specification-standards)
3. [AsyncAPI for Event-Driven APIs](#asyncapi-for-event-driven-apis)
4. [Live API Playgrounds](#live-api-playgrounds)
5. [Automated SDK Generation](#automated-sdk-generation)
6. [Time-to-First-Call Optimization](#time-to-first-call-optimization)
7. [CI/CD Integration](#cicd-integration)
8. [Templates](#templates)

---

## Single Source of Truth

The API specification (OpenAPI for REST, AsyncAPI for event-driven) is the **master record** from which all other artifacts are derived. This is a hard architectural constraint, not a preference.

### The Derivation Chain

```
┌──────────────────┐
│  API Specification│  ◄── This is the ONLY manually edited artifact
│  (OpenAPI/AsyncAPI)│
└────────┬─────────┘
         │
    ┌────▼────┐  ┌─────────┐  ┌───────────┐  ┌──────────┐
    │  Docs   │  │  SDKs   │  │ Playground │  │ Contract │
    │  Site   │  │ (TS,Py, │  │  (Try It)  │  │  Tests   │
    │         │  │  Go...) │  │            │  │  (Prism)  │
    └─────────┘  └─────────┘  └───────────┘  └──────────┘
```

**Why this matters:** When a developer changes an endpoint parameter in the code, they update the spec. The CI pipeline regenerates the docs, SDKs, and playground from the updated spec. Drift between documentation and implementation becomes structurally impossible because the pipeline enforces consistency. If the spec is wrong, everything is visibly wrong — and that is the correct failure mode, because a single visible error is better than silent inconsistency across multiple manually-maintained artifacts.

---

## OpenAPI Specification Standards

Use OpenAPI 3.1+ for all REST APIs. Every specification must include:

### Required Elements

| Element | Purpose | Impact |
|---------|---------|--------|
| `info.title` | Human-readable API name | Appears in docs site header |
| `info.description` | API overview in Markdown | Landing page content for the docs |
| `info.version` | Semantic version of the API | SDK versioning, changelog generation |
| `servers` | Base URLs for each environment | Playground environment selector |
| `paths` | All endpoints with parameters, request/response schemas | Core documentation content |
| `components.schemas` | Reusable data models | SDK type generation, validation |
| `components.securitySchemes` | Authentication methods | Playground auth injection |
| `tags` | Logical grouping of endpoints | Documentation navigation structure |

### Schema Quality Rules

- **Every schema has a `description`:** Fields without descriptions force developers to guess their purpose. A schema named `status` with possible values `active`, `pending`, `archived` must describe what each value means.
- **Use `examples` liberally:** Every request body, response body, and parameter includes at least one realistic example. Examples power the playground's pre-filled request forms.
- **Enumerate explicitly:** When a field has a fixed set of values, use `enum`. Never rely on prose descriptions like "one of: active, pending, archived" when the spec can enforce it.
- **Mark required fields:** Every object schema explicitly marks which properties are `required`. Missing `required` annotations lead to SDK type systems that allow invalid requests.

### Endpoint Documentation

Every path operation includes:

```yaml
/meetings/{meetingId}:
  get:
    operationId: getMeeting
    summary: Retrieve a meeting by ID
    description: |
      Returns the full meeting resource, including participants,
      agenda items, and transcription status.

      **Permissions:** Requires `meetings:read` scope.
    tags:
      - Meetings
    parameters:
      - name: meetingId
        in: path
        required: true
        description: Unique identifier of the meeting (UUID format)
        schema:
          type: string
          format: uuid
        example: "550e8400-e29b-41d4-a716-446655440000"
    responses:
      "200":
        description: Meeting found
        content:
          application/json:
            schema:
              $ref: "#/components/schemas/Meeting"
            example:
              id: "550e8400-e29b-41d4-a716-446655440000"
              title: "Sprint Planning"
              status: "scheduled"
      "404":
        description: Meeting not found
        content:
          application/json:
            schema:
              $ref: "#/components/schemas/Error"
```

---

## AsyncAPI for Event-Driven APIs

Use AsyncAPI 3.0+ for event-driven APIs (WebSocket, SSE, Kafka, Pub/Sub). AsyncAPI serves the same single-source-of-truth role for asynchronous systems that OpenAPI serves for REST.

### Key Elements

| Element | Purpose |
|---------|---------|
| `channels` | Named communication paths (topics, queues, streams) |
| `operations` | What actions can be performed (send, receive) on each channel |
| `messages` | Payload schemas with headers and content types |
| `components.schemas` | Reusable data models (shared with OpenAPI where possible) |

### Example

```yaml
asyncapi: 3.0.0
info:
  title: Meeting Events API
  version: 1.0.0
  description: Real-time meeting lifecycle events via Server-Sent Events

channels:
  meetingEvents:
    address: /v1/meetings/{meetingId}/events
    messages:
      meetingStarted:
        $ref: "#/components/messages/MeetingStarted"
      transcriptChunk:
        $ref: "#/components/messages/TranscriptChunk"

operations:
  receiveMeetingEvents:
    action: receive
    channel:
      $ref: "#/channels/meetingEvents"
    summary: Subscribe to real-time meeting events

components:
  messages:
    MeetingStarted:
      name: meeting.started
      contentType: application/json
      payload:
        type: object
        properties:
          meetingId:
            type: string
            format: uuid
          startedAt:
            type: string
            format: date-time
```

---

## Live API Playgrounds

A live playground allows developers to test API endpoints directly from the documentation page, with real requests and real responses. The goal is to eliminate the friction between "reading the docs" and "making my first call."

### Architecture

```
┌─────────────┐     ┌──────────────┐     ┌─────────────┐
│  Playground  │────▶│  Backend     │────▶│  API Server  │
│  UI (Browser)│     │  Proxy       │     │  (Target)    │
│              │◀────│  (CORS bypass│◀────│              │
└─────────────┘     │  + Auth)     │     └─────────────┘
                    └──────────────┘
```

### Requirements

- **Auto-injected authentication:** The playground pre-fills authentication tokens (OAuth 2.1 bearer tokens, API keys) from the developer's active session. Developers should not have to copy-paste tokens.
- **CORS proxy:** Browser-based playgrounds cannot make direct cross-origin requests to the API. A backend proxy handles CORS transparently so the developer experience is seamless.
- **Pre-filled examples:** Request bodies are pre-populated with realistic example data from the OpenAPI spec. Developers can modify and submit immediately.
- **Response rendering:** Responses are syntax-highlighted and collapsible. Large payloads show a summary with an expand option.
- **Environment selector:** Developers can switch between sandbox, staging, and production endpoints. Default to the sandbox to prevent accidental production mutations.

### Implementation Options

| Tool | Approach | Best For |
|------|----------|----------|
| **Swagger UI** | Open-source, renders OpenAPI specs as interactive docs | Self-hosted, customizable |
| **Stoplight Elements** | Embeddable React components | Integrated into existing docs sites |
| **Redocly** | Full docs platform with "Try It" panel | Enterprise API portals |
| **Mintlify** | Managed docs platform with built-in playground | Fast setup, low maintenance |

---

## Automated SDK Generation

Client SDKs are generated from the API specification, not written by hand. Manual SDK maintenance is an antipattern that guarantees drift from the spec within days of any API change.

### Generation Pipeline

```
┌────────────┐     ┌───────────────┐     ┌─────────────┐
│  OpenAPI    │────▶│  Generator    │────▶│  SDKs       │
│  Spec       │     │  (Fern,       │     │  (TypeScript,│
│  (updated)  │     │   Stainless,  │     │   Python,    │
│             │     │   openapi-gen)│     │   Go, etc.)  │
└────────────┘     └───────────────┘     └──────┬──────┘
                                                │
                                         ┌──────▼──────┐
                                         │  Published   │
                                         │  to npm,     │
                                         │  PyPI, etc.  │
                                         └─────────────┘
```

### Generator Selection

| Generator | Strengths | Output Quality |
|-----------|-----------|----------------|
| **Fern** | Type-safe, idiomatic SDKs with full docs | Production-grade, manually reviewable |
| **Stainless** | Purpose-built for API companies | Enterprise-grade |
| **openapi-generator** | Widest language support, community-driven | Varies by language template |

### SDK Quality Standards

Generated SDKs must meet these standards before publication:

- **Type safety:** All request/response types are strongly typed. No `any` or `interface{}`.
- **Idiomatic naming:** Method and parameter names follow the target language's conventions (camelCase for TypeScript, snake_case for Python, PascalCase for Go).
- **Error handling:** SDK methods return structured error types, not raw HTTP responses.
- **Authentication:** The SDK handles token injection, refresh, and retry transparently.
- **Documentation:** Every public method includes a docstring generated from the OpenAPI operation's `summary` and `description`.

---

## Time-to-First-Call Optimization

Time-to-first-call (TTFC) is the primary metric for API documentation quality. It measures the elapsed time from a developer's first visit to the docs until they successfully make their first real API call. Every documentation design decision is evaluated against its impact on TTFC.

### TTFC Reduction Strategies

| Strategy | TTFC Impact | Implementation |
|----------|-------------|----------------|
| **Quickstart on landing page** | -60% | The first thing a developer sees is a 3-step quickstart, not a feature overview |
| **Copy-paste code snippets** | -40% | Every endpoint includes copy-ready code in the developer's language |
| **Auto-provisioned sandbox** | -80% | Signing up creates a pre-configured sandbox with test data |
| **Pre-filled playground** | -50% | The "Try It" panel has realistic example data ready to submit |
| **Inline authentication** | -30% | Auth tokens are generated and injected without leaving the docs page |

### The Quickstart Pattern

Every API documentation site opens with a quickstart that follows this structure:

```markdown
## Quickstart

### 1. Get your API key
[One-click link to API key generation]

### 2. Install the SDK
[Language-tabbed install commands: npm, pip, go get]

### 3. Make your first call
[Language-tabbed code snippet that makes a real API call]
[Expected output shown below the snippet]
```

The quickstart must be completable in under 5 minutes. If it takes longer, the onboarding flow has friction that must be removed.

---

## CI/CD Integration

The API documentation pipeline runs on every change to the specification:

### Pipeline Stages

```yaml
# Triggered on changes to openapi.yaml or asyncapi.yaml
api-docs-pipeline:
  stages:
    - validate:
        description: Lint and validate the specification
        tools:
          - spectral: OpenAPI/AsyncAPI linting rules
          - openapi-diff: Breaking change detection
        gate: Fail PR on validation errors or breaking changes without version bump

    - generate-docs:
        description: Render documentation from the specification
        tools:
          - redocly-build: Generate static HTML docs
          - llms-txt-gen: Update llms.txt with new endpoints
        output: docs/ directory

    - generate-sdks:
        description: Generate client SDKs in all supported languages
        tools:
          - fern-generate: TypeScript, Python, Go SDKs
        output: sdk/ directory, published to package registries

    - contract-tests:
        description: Validate the API server against the specification
        tools:
          - prism-mock: Generate mock server from spec
          - prism-proxy: Validate live responses against spec
        gate: Fail if API responses diverge from specification

    - deploy-playground:
        description: Update the live playground with new spec
        tools:
          - swagger-ui-deploy: Update interactive API explorer
```

### Breaking Change Detection

Configure `openapi-diff` or `oasdiff` to detect breaking changes in the specification:

```yaml
# Breaking changes that MUST increment the major version
breaking_changes:
  - Removing an endpoint
  - Removing a required request parameter
  - Changing a response schema (removing fields)
  - Changing authentication requirements

# Non-breaking changes that increment the minor version
non_breaking_changes:
  - Adding a new endpoint
  - Adding an optional parameter
  - Adding new response fields
  - Adding new enum values
```

---

## Templates

### Minimal OpenAPI Spec Skeleton

```yaml
openapi: 3.1.0
info:
  title: "[Project Name] API"
  version: "1.0.0"
  description: |
    [One-paragraph API overview. What it does, who it's for.]
  contact:
    name: "[Team Name]"
    email: "[team@example.com]"

servers:
  - url: https://api.example.com/v1
    description: Production
  - url: https://sandbox.api.example.com/v1
    description: Sandbox

security:
  - bearerAuth: []

tags:
  - name: "[Resource Name]"
    description: "[What this group of endpoints manages]"

paths:
  /[resource]:
    get:
      operationId: list[Resource]
      summary: List all [resources]
      tags: ["[Resource Name]"]
      responses:
        "200":
          description: "[Resource] list retrieved"

components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
```

### SDK Generation Config (Fern)

```yaml
# fern.config.yaml
organization: "[org-name]"
version: "1.0.0"

api:
  path: openapi.yaml

generators:
  - name: fernapi/fern-typescript-node-sdk
    version: 0.x.x
    output:
      location: npm
      package-name: "@[org]/[api]-sdk"

  - name: fernapi/fern-python-sdk
    version: 0.x.x
    output:
      location: pypi
      package-name: "[api]-sdk"

  - name: fernapi/fern-go-sdk
    version: 0.x.x
    output:
      location: github
      repo: "[org]/[api]-go"
```

---

## Versioning & Deprecation

APIs evolve. Versioning ensures existing integrations continue to function while new capabilities are introduced. Deprecation is the communication discipline that gives consumers time to migrate.

### Versioning Strategy

| Strategy | When to Use | Example |
|----------|------------|---------|
| **URL path versioning** | Public APIs, maximum clarity | `/v1/meetings`, `/v2/meetings` |
| **Header versioning** | Internal APIs, cleaner URLs | `Accept: application/vnd.api+json; version=2` |

URL path versioning is the default for external-facing APIs because it is the most visible and least surprising convention for consumers.

### Multi-Version Documentation

When multiple API versions coexist, each version has its own documentation section:

```
docs/api/
├── v1/                    # Full reference for v1 (deprecated)
│   ├── overview.md
│   └── endpoints.md
├── v2/                    # Full reference for v2 (current)
│   ├── overview.md
│   └── endpoints.md
└── migration-v1-to-v2.md  # Migration guide
```

**Version labels:** Every version-specific page displays a clear label:
- **Latest** — The current, recommended version
- **Stable** — The previous major version, still fully supported
- **Deprecated** — Functional but scheduled for removal (display a banner with sunset date)
- **Sunset** — No longer available (display a redirect to the migration guide)

### Deprecation Communication

Deprecation is a lifecycle event that requires proactive, multi-channel communication:

**Timeline:**
1. **Announcement** (6+ months before sunset) — Deprecation notice in docs, release notes, changelog
2. **Active warnings** (3+ months before sunset) — Response headers, dashboard banners, email to API consumers
3. **Final warning** (1 month before sunset) — Elevated warnings in all channels
4. **Sunset** — Endpoints return `410 Gone` with migration guide link

**In documentation:**
```markdown
> ⚠️ **Deprecated:** This endpoint is deprecated and will be removed
> on YYYY-MM-DD. Use [POST /v2/meetings](/api/v2/meetings) instead.
> See the [Migration Guide](../migration-v1-to-v2.md).
```

### The Sunset HTTP Header

Use the `Sunset` header (RFC 8594) to programmatically notify API consumers that an endpoint is deprecated:

```http
HTTP/1.1 200 OK
Sunset: Sat, 01 Nov 2026 00:00:00 GMT
Deprecation: true
Link: <https://docs.example.com/migration-v2>; rel="sunset"
```

This allows automated tooling and SDK wrappers to detect deprecation without requiring human monitoring of documentation pages.

### Deprecation in OpenAPI Specs

Mark deprecated operations directly in the specification:

```yaml
/v1/meetings:
  get:
    deprecated: true
    x-sunset-date: "2026-11-01"
    summary: "[DEPRECATED] List meetings — use /v2/meetings"
    description: |
      **Deprecated:** This endpoint will be removed on 2026-11-01.
      Migrate to [GET /v2/meetings](#operation/listMeetingsV2).
```

