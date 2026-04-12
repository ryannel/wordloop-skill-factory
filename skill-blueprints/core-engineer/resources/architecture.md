---
trigger: always_on
---

# Architecture Rule: Clean Architecture for Word Loop Core

## 1. Context & Scope

This rule applies to all Go code generation and refactoring. While project-level files exist in `cmd/` and `scripts/`, the core application logic is strictly confined to the `internal/` directory. This ensures implementation details cannot be imported by external packages.

## 2. Architectural Layers & Dependency Rules

**Dependencies must only point INWARD.** Inner layers must never import from outer layers.

### Inner Layers (ordered inward)

* **Domain (`internal/core/domain`):**
  * **Purpose:** Business entities, sentinel errors, and core domain logic.
  * **Dependency:** Zero dependencies. Pure Go only.

* **Gateways (`internal/core/gateway`):**
  * **Purpose:** Repository interfaces (contracts) for external data.
  * **Dependency:** Depends only on Domain.

* **Services (`internal/core/service`):**
  * **Purpose:** Use-case orchestration and business logic.
  * **Dependency:** Depends on Domain and Gateways.

### Outer Adapters (siblings — same depth, different direction)

Providers and Entrypoints are both outer-layer adapters. **Neither imports the other.** They sit on opposite sides of the hexagon: one drives inbound requests, the other fulfills outbound dependencies.

* **Providers (`internal/provider`):**
  * **Purpose:** Concrete implementations (Postgres, Redis, S3). Driven (outbound) adapters.
  * **Dependency:** Depends on Domain and Gateways. **Never imports Service or Entrypoints.**

* **Entrypoints (`internal/entrypoints`):**
  * **Purpose:** API routes, middleware, and request/response marshaling. Driving (inbound) adapters.
  * **Dependency:** Depends on Domain and Services. **Never imports Providers or Gateways directly.**

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
                │   (Postgres, Redis, S3, external APIs)    │
                │         Driven / Outbound Adapter         │
                └──────────────────────────────────────────┘

    Entrypoints and Providers are sibling adapters. Neither imports
    the other. Only cmd/ (the wiring layer) has visibility into both.
```

## 3. Domain Integrity & Error Handling

* **Encapsulate Logic:** If a rule only requires data from within a single struct, it MUST be a method on that struct (e.g., `meeting.IsUpcoming()`).
* **Self-Validation:** Entities must ensure consistency via constructors (`NewMeeting(...)`) or `Validate()` methods.
* **Sentinel Errors:** Define "Expected" errors (e.g., `ErrNotFound`) in the Domain layer.
* **Error Wrapping:** Providers must wrap driver errors (e.g., `sql.ErrNoRows`) using `%w` to maintain context while satisfying Domain error checks.
* **Purity:** Domain methods must be side-effect-free. No database calls or network I/O.

## 4. The "Gold Thread" (Context & DI)

* **Context Propagation:** `context.Context` must be the first argument of every function in the Service, Gateway, and Provider layers. **Never** store context in a struct.
* **Constructor Injection:** Every struct must have a `New...` function. Return concrete types, but accept interfaces as arguments.
* **No Globals:** Global variables for database pools, loggers, or configuration are strictly forbidden. Inject them at the entrypoint (`cmd/`).

## 5. Database & Migrations

* **Never Manual:** All schema changes must be migration files in `scripts/migrations/`.
* **Naming:** Use `YYYYMMDDHHMMSS_description.up.sql` and `.down.sql`.
* **Transactions:** The Service layer defines transaction boundaries; Providers execute the logic.
* **Idempotency:** Use `IF NOT EXISTS` for safety.

## 6. Testing Strategy
Please ensure that tests are updated with every code change. This should cover all layers of the application.

* **Service Layer:** Use `internal/core/gateway/mocks`. Test orchestration logic only.
* **Provider Layer:** **NO MOCKS.** Test against a real Postgres instance (e.g., via Docker/testcontainers) using `testdb_setup_test.go`.
* **Entrypoints:** Use `httptest` and `internal/core/service/mocks`. Test status codes and JSON mapping.

## 7. Hard Rules

* **Do not** use `init()` functions.
* **Do not** use `panic()`. Always return and handle errors.
* **Do not** return interfaces; return concrete structs (e.g., `func NewService(...) *Service`).
* **Do not** use `fmt.Print`. Use a structured logger injected via Context or DI.
* **Do not** leak HTTP or SQL concepts into the Service/Domain layers.
* **Do not** import Providers from Entrypoints, or Entrypoints from Providers. They are sibling adapters.