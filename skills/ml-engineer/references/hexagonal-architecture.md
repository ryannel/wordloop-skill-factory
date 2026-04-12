# Hexagonal Architecture (`wordloop-ml`)

The `wordloop-ml` engine strictly relies on clean architectural dependency inversion. Dependencies must always point **INWARD** toward core business logic. Outward direction leakage is deeply prohibited.

## Core Layers
Map all feature modules inside specific bound directories (e.g. `src/wordloop/core/transcription/`).

### 1. Domain (`src/wordloop/core/domain`)
Business entities bound strictly by `dataclasses` or `Pydantic v3`. It has zero dependencies to outer frameworks. It may utilize logic libraries assuming they don't break I/O.
*   **Rule:** Maintain pure application exceptions here (`class TranscriptionFailedError(Exception)`) so outer providers can map back to semantic domain failures.

### 2. Gateways (`src/wordloop/core/gateways`)
Defines the outbound interaction capability. 
*   **Rule:** NEVER use `abc.ABC`. Explicitly utilize `typing.Protocol` for duck-typed structural subtyping. 
*   **Rule:** Gateway signatures must exclusively ingest and return mapped `Domain` structures, completely blinding the services to SDK footprints (do not accept an `openai.ChatCompletion` object; accept an `AIResponse` Pydantic Domain Model).

### 3. Services (`src/wordloop/core/services`)
Use-case orchestration. Takes inbound data from Entrypoints, executes Domain logic, and fires commands through Gateway Protocols.
*   **Constructor Injection:** Accept `Protocol` definitions in the `__init__` constructor explicitly.

## The Sibling Adapters (Outer Ring)
These two components live outside the Core. They never import each other. Only the top-level Dependency Wiring layer connects them together.

### 4. Providers (`src/wordloop/providers`)
"Driven" outbound adapters. These concretely implement the `typing.Protocol` definitions defined in the Gateways.
*   **Mapping:** Always map external SDK data down into standard Domain schemas before returning into the boundary.
*   **Error Wrapping:** Catch `botocore` or `asyncpg` network disconnects and cleanly raise the exact semantic `Domain` exception natively defined in the interior.

### 5. Entrypoints (`src/wordloop/entrypoints`)
"Driving" inbound adapters. These contain FastMCP integrations, FastAPI Routers, or CLI invocations triggering the `Services`. They perform pure inbound schema validation but make zero active business decisions.
