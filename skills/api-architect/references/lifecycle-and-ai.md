# Lifecycle Evolution & AI Consumption

Designing an API platform requires committing to a permanent operational product strategy. You must maintain stability for existing integrators while establishing clear semantic highways designed specifically for AI agent integration.

## Date-Based Versioning

Major software platforms have abandoned simple v1, v2 string definitions in favor of granular **Date-Based Versioning**. 
* Pin integrations to exact dates (e.g., `Stripe-Version: 2026-03-25`). 
* The core server processes logic against the newest codebase version, running the outbound representation response backwards through localized "Downgrade Transformer" modules until it matches the physical layout standard documented on the client’s version date.

## Decommissioning and EOL
Provide clear runways for Integrators avoiding blind 404 breaks:
1. **RFC 8594 `Deprecation` Header:** Warn the integrator that the resource route is actively deprecated, triggering warnings in their HTTP logging observability platforms without breaking their execution. Include a Link header pointing to migration docs.
2. **RFC 8594 `Sunset` Header:** Provides a strict, machine-readable Unix timestamp mapping indicating exactly when the endpoint will fail.
3. **HTTP 410 Gone:** Once sunset passes, transition away from `404 Not Found` to a strict `410 Gone`. This differentiates intentional service termination from accidental typographic errors in standard reporting.

## Machine Discovery (`llms.txt`)
With 30% of API bandwidth dedicated to LLM and autonomous processing agents, your platform requires direct semantic optimization.

Publish an `llms.txt` file mounted directly at the root platform directory (`yoursite.com/llms.txt`).
* Keep it under 3000 tokens. 
* Direct the crawler directly to `.md` versions of OpenAPI documentation, minimizing HTML parsing noise.
* Organize linking trees specifically aligned with AI functional execution paths.

## Cross-Boundary TraceContext
Maintain distributed monitoring parity on requests spanning synchronous boundary hops and deeply nested asynchronous microservices.
Ensure gateways and code generate OpenTelemetry W3C **`traceparent`** and **`tracestate`** variables, injecting them within HTTP request headers (`traceparent: 00-0af7651916cd43dd8448eb211c80319c-b7ad6b7169203331-01`). Carry this identical hex context physically through any asynchronous event boundary payloads to trace exact node failure pipelines reliably.
