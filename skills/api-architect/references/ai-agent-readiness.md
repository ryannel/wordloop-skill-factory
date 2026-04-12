# AI-Agent Readiness & Machine-Readable API Design

## Table of Contents

- [Why AI-Agent Readiness Matters](#why-ai-agent-readiness-matters)
- [Tool-Friendly Schema Design](#tool-friendly-schema-design)
- [Operation Naming for AI Agents](#operation-naming-for-ai-agents)
- [Comprehensive Examples](#comprehensive-examples)
- [The llms.txt Specification](#the-llmstxt-specification)
- [Developer Portal Design](#developer-portal-design)
- [Contract Testing and Drift Prevention](#contract-testing-and-drift-prevention)
- [Observability by Design](#observability-by-design)

---

## Why AI-Agent Readiness Matters

A significant and growing portion of API consumption comes from AI agents and LLM-powered tools rather than human developers. These agents discover, understand, and call APIs autonomously based on structured metadata. APIs that lack machine-readable semantics — vague operation names, missing examples, undocumented edge cases — cause agent hallucinations, incorrect parameter mapping, and failed integrations.

Designing for AI-agent readiness improves the experience for human developers too. The same properties that help an LLM understand your API (clear naming, comprehensive examples, structured schemas) are the properties that reduce human onboarding time.

---

## Tool-Friendly Schema Design

AI agents reason about API capabilities by reading OpenAPI specifications. The quality of the specification directly determines how reliably an agent can map user intent to API calls.

### Descriptive, Semantic Naming

Every operation, schema, and property needs names that convey meaning without requiring context:

```yaml
# Good — self-explanatory
operationId: list_users_by_department
operationId: get_user_by_email
operationId: create_payment_intent
operationId: cancel_subscription_immediately

# Bad — ambiguous
operationId: getUsers
operationId: fetch
operationId: process
operationId: handle
```

### Rich Descriptions

Every parameter, property, and response must have a `description` field. Agents use descriptions to decide which parameters to fill and how to interpret responses:

```yaml
parameters:
  - name: status
    in: query
    description: >
      Filter users by account status. Accepts one or more comma-separated
      values. Active users have completed onboarding and can access all
      features. Suspended users are temporarily locked and cannot log in.
    schema:
      type: string
      enum: [active, inactive, suspended]
```

### Explicit Constraints

Agents need to know valid ranges, patterns, and formats to generate correct requests:

```yaml
properties:
  email:
    type: string
    format: email
    description: User's primary email address. Must be unique across the platform.
    example: "user@example.com"
  age:
    type: integer
    minimum: 13
    maximum: 150
    description: User's age in years. Minimum age of 13 required for account creation.
  username:
    type: string
    minLength: 3
    maxLength: 30
    pattern: "^[a-zA-Z0-9_-]+$"
    description: >
      Public username. 3-30 characters, alphanumeric with underscores and
      hyphens. Cannot be changed after initial registration.
```

---

## Operation Naming for AI Agents

The `operationId` is the primary handle an AI agent uses to identify and invoke an API operation. Follow these conventions:

### Naming Pattern

```
{verb}_{resource}[_by_{field}][_with_{qualifier}]
```

### Examples

| Operation | operationId |
|----------|-------------|
| List all users | `list_users` |
| Get user by ID | `get_user_by_id` |
| Get user by email | `get_user_by_email` |
| Create a new order | `create_order` |
| Update a user's profile | `update_user_profile` |
| Delete an API key | `delete_api_key` |
| Search products | `search_products` |
| Activate a subscription | `activate_subscription` |
| List orders for a specific user | `list_user_orders` |
| Get payment intent status | `get_payment_intent_status` |

### Why This Matters

When an LLM reads a list of available operations, it performs semantic matching between the user's natural language request and the operation names. `get_user_by_email` maps unambiguously to "find the user with email john@example.com." `getUser` requires the agent to guess whether it accepts an ID, email, or username.

---

## Comprehensive Examples

Examples are the single most valuable piece of metadata for AI agents. An agent that sees a concrete request/response pair can generate correct API calls with much higher reliability than one working from schema definitions alone.

### Request Examples

```yaml
requestBody:
  required: true
  content:
    application/json:
      schema:
        $ref: "#/components/schemas/CreateUserRequest"
      examples:
        minimal:
          summary: Create user with required fields only
          value:
            email: "jane@example.com"
            name: "Jane Smith"
        full:
          summary: Create user with all optional fields
          value:
            email: "jane@example.com"
            name: "Jane Smith"
            metadata:
              department: "engineering"
              title: "Staff Engineer"
```

### Response Examples

```yaml
responses:
  "200":
    description: User found
    content:
      application/json:
        examples:
          active_user:
            summary: Active user with complete profile
            value:
              id: "usr_abc123"
              email: "jane@example.com"
              name: "Jane Smith"
              status: "active"
              created_at: "2024-01-15T10:30:00Z"
              metadata:
                department: "engineering"
```

### Error Examples

```yaml
responses:
  "422":
    description: Validation failed
    content:
      application/problem+json:
        examples:
          invalid_email:
            summary: Invalid email format
            value:
              type: "https://api.example.com/errors/validation-error"
              title: "Validation Error"
              status: 422
              detail: "Request validation failed."
              errors:
                - pointer: "/email"
                  code: "INVALID_FORMAT"
                  detail: "Must be a valid email address."
```

---

## The llms.txt Specification

The `llms.txt` file acts as a structured entry point for AI crawlers, helping LLMs navigate your documentation site efficiently. It is hosted at the root domain as a plain text file.

### Location and Format

- **URL**: `https://yoursite.com/llms.txt`
- **Content-Type**: `text/plain`
- **Format**: Markdown with structured sections
- **Size**: Under 10KB / 3,000 tokens to fit within an agent's context window

### Template

```markdown
# Example API

> Example API provides user management, order processing, and payment
> handling for e-commerce platforms. It uses REST with OpenAPI 3.1
> specifications and OAuth 2.1 authentication.

## API Reference

- [OpenAPI Specification](https://api.example.com/docs/openapi.yaml)
- [API Reference](https://docs.example.com/api-reference.md)
- [Authentication Guide](https://docs.example.com/auth.md)

## Guides

- [Getting Started](https://docs.example.com/getting-started.md)
- [Pagination](https://docs.example.com/pagination.md)
- [Error Handling](https://docs.example.com/errors.md)
- [Webhooks](https://docs.example.com/webhooks.md)

## SDKs

- [TypeScript SDK](https://docs.example.com/sdk/typescript.md)
- [Python SDK](https://docs.example.com/sdk/python.md)
- [Go SDK](https://docs.example.com/sdk/go.md)
```

### Best Practices

- **Link to `.md` versions** of documentation to avoid HTML noise that wastes tokens
- **Group links by user journey** — API reference, guides, SDKs
- **Prioritize high-value links** — The most important links go first
- **Keep the summary concise** — 1-2 sentences in the blockquote
- **Update when the API changes** — Treat `llms.txt` as a first-class artifact

---

## Developer Portal Design

The developer portal is the complete hub for API consumers. It should provide:

### Interactive Documentation

- **Generated from OpenAPI spec** — Never manually authored
- **Try-it-out playground** — Test endpoints with real credentials
- **Code snippets** — Auto-generated in multiple languages from examples
- **Authentication helper** — Inline token generation for testing

### Content Structure

| Section | Purpose |
|---------|---------|
| Getting Started | Time-to-first-call in under 5 minutes |
| API Reference | Interactive, auto-generated from OpenAPI |
| Guides | Pagination, error handling, webhooks, auth |
| SDKs | Per-language installation and usage |
| Changelog | Versioned list of changes |
| Status Page | Real-time API health and incident history |

### Analytics

Track which pages developers visit, which endpoints they test, and where they drop off. This data drives documentation improvements and identifies confusing API surfaces.

---

## Contract Testing and Drift Prevention

### CI/CD Integration

Run automated contract validation on every pull request:

```bash
# Lint the spec for errors and style violations
npx @redocly/cli lint openapi.yaml

# Diff the spec against the previous version
npx @redocly/cli diff openapi-previous.yaml openapi.yaml

# Run contract tests against the implementation
npx @stoplight/prism-cli proxy openapi.yaml http://localhost:3000 --errors
```

### Breaking Change Detection

Automated workflows should flag:
- Removed endpoints or fields
- Changed field types
- New required fields in request bodies
- Modified response structure
- Changed status codes

These checks run before merge, preventing accidental breaking changes from reaching production.

### SDK Generation Pipeline

Never maintain SDKs manually. Generate them from the OpenAPI spec:

```
OpenAPI Spec → Code Generator → SDK Package → Published to Registry
```

This ensures documentation, SDKs, and implementation are always in sync, eliminating "drift" — the divergence between what the docs say and what the API actually does.

---

## Observability by Design

### Structured Logging

Log every API call with structured metadata:

```json
{
  "timestamp": "2024-01-15T10:30:00Z",
  "request_id": "req_abc123",
  "method": "GET",
  "path": "/users/123",
  "status": 200,
  "duration_ms": 45,
  "user_id": "usr_xyz789",
  "client_type": "sdk-typescript-2.1.0"
}
```

Never log sensitive data (auth tokens, passwords, PII beyond what's required for debugging).

### Monitoring Metrics

Track at minimum:
- **Latency** — p50, p95, p99 per endpoint
- **Error rate** — 4xx and 5xx percentages
- **Request volume** — Requests per second per endpoint
- **Availability** — Uptime percentage against SLA

### Distributed Tracing

Include trace context headers (`traceparent` from W3C Trace Context) across service boundaries. This enables end-to-end request tracing through microservice architectures.

```http
traceparent: 00-abc123def456-789012-01
```

### Alerting

Set alerts on:
- Latency regression (p99 exceeds SLA threshold)
- Error rate spike (5xx rate exceeds baseline)
- Rate limit exhaustion patterns (specific clients hitting limits repeatedly)
- Contract drift (spec validation failures in CI)
