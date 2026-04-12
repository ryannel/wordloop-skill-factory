---
trigger: always_on
---

# Architecture Rule: Clean Architecture for WordLoop Python Core

## 1. Context & Scope

This rule applies to all Python code generation and refactoring within `src/wordloop`. The core application logic must remain in `src/wordloop/core/`. External integrations belong in `src/wordloop/providers/`.

## 2. Architectural Layers & Dependency Rules

**Dependencies must only point INWARD.** Inner layers must never import from outer layers.

### Inner Layers (ordered inward)

* **Domain (`src/wordloop/core/domain`):**
  * **Purpose:** Business entities and core logic using `dataclasses` or `Pydantic`.
  * **Zero-Dependency Core:** Standard library and Pydantic/Dataclasses only. No I/O.
  * **No Implementation Tags:** Models must not contain database-specific decorators or library-specific types (e.g., no `SQLAlchemy` ORM models here).
  * **Universal Vocabulary:** Define core application exceptions (`src/wordloop/core/exceptions.py`) and constants here.
  * **Testing:** Pure Unit Tests. Verify state transitions and business rules with zero mocks.

* **Gateways (`src/wordloop/core/gateways`):**
  * **Purpose:** `typing.Protocol` or `abc.ABC` definitions that define **capabilities**.
  * **Contractual Masters:** Gateways define _what_ (e.g., `store`), never _how_.
  * **No Leaky Abstractions:** Signatures must use **Domain** entities. Never reference SDK types (e.g., `openai.ChatCompletion`) or transport types.
  * **The Golden Rule:** Use generic names. `publish(msg: Message)`, not `send_to_sqs(msg: Message)`.

* **Services (`src/wordloop/core/services`):**
  * **Purpose:** Use-case orchestration. This is where the application "decides" what happens.
  * **Dependency Injection:** Services depend on **Gateways** (Protocols/ABCs), not concrete Providers.
  * **Protocol Consumer Rule:** Services should return concrete Domain objects. If a Service is used by an Entrypoint, the **Entrypoint** defines the Protocol/Interface it requires from the Service.
  * **Transaction Boundaries:** Coordinate workflows by fetching data, applying domain logic, and persisting results.

### Outer Adapters (siblings — same depth, different direction)

Providers and Entrypoints are both outer-layer adapters. **Neither imports the other.** They sit on opposite sides of the hexagon: one drives inbound requests, the other fulfills outbound dependencies.

* **Providers (`src/wordloop/providers/`):**
  * **Purpose:** Concrete implementations of Gateway interfaces (The "Adapter").
  * **Mapping (Domain Alignment):** Translates external SDK responses into **Domain Entities**.
  * **Error Wrapping:** Catch library-specific errors (e.g., `botocore.exceptions.ClientError`) and raise a corresponding **Core Exception** defined in the Domain layer.
  * **Testing:** Integration Tests only. Use `testcontainers-python` to verify actual I/O against real instances.

* **Entrypoints (`src/wordloop/entrypoints/`):**
  * **Purpose:** The interaction layer (FastAPI, CLI).
  * **No Business Logic:** Entrypoints validate inputs, map request schemas to Domain objects, and call a Service. They do not make business decisions.
  * **Testing:** Use `fastapi.testclient.TestClient` with mocked Services to verify routing and status codes.

### Dependency Diagram

```
                ┌──────────────────────────────────────────┐
                │              ENTRYPOINTS                  │
                │    (HTTP handlers, middleware, routing)    │
                │         Driving / Inbound Adapter         │
                └──────────────┬───────────────────────────┘
                               │ depends on
                ┌──────────────▼───────────────────────────┐
                │              SERVICES                     │
                │        (Use-case orchestration)           │
                │    ┌─────────────────────────────────┐   │
                │    │           GATEWAYS               │   │
                │    │     (Repository interfaces)      │   │
                │    │    ┌─────────────────────────┐   │   │
                │    │    │         DOMAIN           │   │   │
                │    │    │      (Pure logic)        │   │   │
                │    │    └─────────────────────────┘   │   │
                │    └─────────────────────────────────┘   │
                └──────────────▲───────────────────────────┘
                               │ depends on
                ┌──────────────┴───────────────────────────┐
                │              PROVIDERS                    │
                │    (Postgres, Redis, LLMs, Vector DB)     │
                │         Driven / Outbound Adapter         │
                └──────────────────────────────────────────┘

    Entrypoints and Providers are sibling adapters. Neither imports
    the other. Only the container/wiring layer has visibility into both.
```


---

## 3. Dependency Injection & State

- **Constructor Injection:** Use `__init__` for all dependencies.

- **No Globals:** Do not use global database clients. Initialize them in the entrypoint startup (e.g., `lifespan` in FastAPI) and inject them.

- **Wiring:** All concrete Provider-to-Service wiring happens at the outermost edge (the Entrypoint or a dedicated `container.py`).


---

## 4. Integrity & System Testing

## **Bootstrap Verification (The "Smoke" Test)**

To ensure the application is wired correctly:

- **Wiring Test:** A test in `tests/system/test_bootstrap.py` that attempts to initialize the full dependency tree.

- **Validation:** Ensures that all required environment variables are present and that the DI container (or manual wiring) doesn't fail on startup.


## **Golden Path System Tests**

- **Location:** `tests/system/`.

- **Strategy:** Run a live instance of the app (e.g., using `uvicorn` in a subprocess or `TestClient` with real providers) against real infrastructure via `testcontainers`.

- **Zero Mocks:** These tests verify the "Golden Thread" from the API route all the way to the database/third-party SDK.

- **Scope:** Focus strictly on high-value success paths.


---

## 5. Hard Rules

1. **Pydantic Everywhere:** Use Pydantic for all data boundaries (Request/Response and Domain).

2. **No `print()`:** Use structured logging.

3. **No Circular Imports:** If an import cycle occurs, it is a sign of leaked layer responsibilities.

4. **No Transport Leakage:** Do not allow FastAPI `Depends`, `Request`, or SQL `Session` objects to enter the Service or Domain layers.

5. **Clean Containers:** Always ensure `container.stop()` or similar cleanup is called in `pytest` fixtures to prevent resource leaks in CI.
