# Hexagonal Architecture (Ports and Adapters)

Golang services in 2026 prioritize deep isolation to ensure code value survives the rapid churn of AI-integrated boundary tools and dependencies. The codebase strictly abides by the "Dependency Rule": dependencies must always point *inward* toward core business logic.

## The Layer Definitions

Architecturally, the codebase strictly maps to the `/internal/` path. No other module should be permitted internal dependency access.

### 1. The Core (Inner Hexagon)
* **Domain (`internal/core/domain`)**: Holds business entities, sentinel errors, and core validation logic. It must never depend on any other application layer (no Services, Providers, or Entrypoints). **Note:** While it must not depend on application layers, it does not strictly have to be "pure go"—it is permitted to use helpful external libraries assuming they don't break the boundary logic constraints.
* **Gateways (`internal/core/gateway`)**: Outbound abstract interfaces defining what the domain needs to function (e.g., `OrderRepository`, `LLMClient`). They allow the business logic to rely on contracts, not concrete providers.
* **Services (`internal/core/service`)**: Use-case orchestration. These depend on the Domain and Gateways, coordinating flows between entities without directly interacting with HTTP or SQL structures.

### 2. The Adapters (Outer Hexagon)
Adapters connect the core to the real world. They are sibling components. They sit on opposite sides of the hexagon. **They must never import one another.**

* **Entrypoints (`internal/entrypoints`)**: "Driving" adapters. They initiate action. These contain HTTP REST handlers, gRPC servers, middleware, or GraphQL/MCP controllers. They parse inbound traffic into strong types and invoke Application Services.
* **Providers (`internal/provider`)**: "Driven" adapters. They fulfill Gateways. These contain concrete logic like PostgreSQL abstractions, Pinecone queries, Redis pipelines, or third-party downstream HTTP integrations. They are the only layers allowed to run native SQL commands.

## Construction and Dependency Injection (DI)
The system is bound together strictly through Context definitions flowing continuously through the golden thread. There are no global config definitions, database pools, or random logger packages permitted structurally. 

Construct structs using `New[Struct]()` initializers returning concrete pointer structs `*Service`, but accepting inbound Gateway interface arguments mapping concrete `internal/provider` logic into abstract `internal/core/gateway` constraints smoothly during application `/cmd/` bootstrapping.

## Exposing Logic securely via MCP (Model Context Protocol) 
AI agent integration utilizes a strictly defined Entrypoint Provider. Treat MCP interfaces structurally similarly to REST connections.
Build an MCP driving adapter to safely parse the inbound agentic queries using type-safe JSON Schemas, execute any pre-approval checks or "Governance Logic" natively, and marshal the data into the Application Service layer to interact with the core safely.
