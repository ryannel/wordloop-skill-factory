---
name: real-time-architect
description: >
  Architect and review Wordloop real-time systems using canonical real-time,
  integration, event-reference, and frontend docs. Use for WebSockets, SSE,
  WebTransport, Pub/Sub, Kafka, optimistic UI, echo suppression, source-
  aware events, multi-device sync, event replay, fan-out, backplanes, live
  updates, CRDTs, CloudEvents, distributed tracing for events, idempotency,
  backpressure, graceful draining, or real-time security. Make sure to use
  this skill whenever a user is working on streaming, evented communication,
  live data synchronization, or real-time protocol decisions, even if they
  don't explicitly ask for a "real-time architect."
---

# Real-Time Architect

Real-time execution architect for streaming and evented systems. This skill guides protocol selection, event contract design, state synchronization, and resilience patterns — choosing the simplest approach that meets consistency and latency requirements.

## Operating Contract

1. Choose the simplest real-time pattern that satisfies consistency and latency needs. SSE before WebSockets. WebSockets before WebTransport. Source-aware streaming before CRDTs. Complexity is justified only by concrete requirements.
2. Use AsyncAPI/event references as the source of truth for event names and payloads. Events are contracts — treat them with the same rigor as REST APIs.
3. Design echo suppression, idempotency, replay, and failure recovery explicitly. These are not edge cases; they are core to correctness in distributed real-time systems.
4. Keep protocol policy in docs and use this skill for routing and verification.

## Core Pillars

1. **Simplest Protocol That Fits** — Real-time has a complexity ladder: polling → SSE → WebSockets → WebTransport → custom protocols. Start at the simplest rung that meets the actual requirements. SSE handles most read-only streaming. WebSockets add bidirectionality. WebTransport adds multiplexed streams. Moving up the ladder increases implementation, debugging, and operational cost. The temptation to reach for the most powerful tool is strong; resist it unless the requirements demand it.

2. **Events as Contracts** — Every event has a name, a versioned schema, delivery semantics, and ordering guarantees. AsyncAPI specs define these contracts the same way OpenAPI defines REST. Event names, payload shapes, and topic structures are not implementation details — they are public interfaces that consumers depend on. Changing them requires the same lifecycle rigor as changing an API.

3. **Source-Aware State Sync** — The server is the source of truth. Clients apply optimistic updates for responsiveness, then reconcile with server-confirmed state. Echo suppression prevents the user's own mutation from appearing twice. Source-aware events carry enough metadata (origin ID, sequence number, timestamp) for clients to merge server state without conflict. This pattern handles the vast majority of collaborative scenarios without CRDTs.

4. **Resilience by Design** — Network interruptions are not exceptional; they are routine. Every real-time connection needs explicit reconnection logic, event replay from a known position, duplicate detection via idempotency keys, and graceful degradation when the backplane is unavailable. These mechanisms are designed upfront, not patched in after the first production outage.

5. **Observable Event Flow** — Distributed event systems are hard to debug without end-to-end tracing. Every event carries a correlation ID. Producers, brokers, and consumers emit trace spans. Dead-letter queues capture unprocessable events for inspection. Backpressure and fan-out metrics are monitored. When an event goes missing, the traces should reveal where it was lost.

## How to Use This Skill

Match the user's task to the smallest relevant reference set. Most tasks touch one or two references.

| Topic | Reference | Load When |
|-------|-----------|-----------|
| Protocol Selection | `references/protocols.md` | Choosing between SSE, WebSockets, WebTransport, or polling. |
| System Design | `references/system-design.md` | Architecting real-time data flow, topology, scaling, and backplane design. |
| Patterns | `references/patterns.md` | Common real-time patterns: fan-out, pub/sub, request-reply, event sourcing. |
| Optimistic UI & Echo Suppression | `references/optimistic-echo-suppression.md` | Client-side optimistic updates, server reconciliation, echo filtering. |
| State Management | `references/state.md` | Distributed state, CRDTs, sequence numbers, conflict resolution. |
| Event Contracts | `references/contracts.md` | AsyncAPI specs, CloudEvents envelope, event naming, versioning. |
| Serialization | `references/serialization.md` | Payload formats (JSON, Protobuf, MessagePack), schema evolution. |
| Resiliency | `references/resiliency.md` | Reconnection, replay, idempotency, backpressure, graceful draining. |
| Observability | `references/observability.md` | Distributed tracing for events, correlation IDs, dead-letter monitoring. |
| Security | `references/security.md` | Auth for persistent connections, token refresh, channel authorization. |
| Client Patterns | `references/client.md` | Frontend WebSocket management, connection lifecycle, state hooks. |
| MCP Integration | `references/mcp.md` | Real-time patterns for MCP tool/resource interactions. |
| gRPC & GraphQL | `references/grpc-graphql.md` | gRPC streaming, GraphQL subscriptions, protocol-specific patterns. |
| Testing | `references/testing.md` | Testing real-time systems: connection lifecycle, event ordering, failure modes. |

## Required Wordloop Context

Read the relevant canonical docs before making non-trivial recommendations or code changes. Prefer the Wordloop docs MCP tools when available; otherwise read the local MDX files under `services/wordloop-docs/content/docs/`.

- `principles/system-design/real-time` — Wordloop's real-time design philosophy and pattern selection guidance.
- `principles/system-design/integration-patterns` — Service-to-service communication patterns.
- `reference/events/core-ws` — WebSocket event reference (source of truth for WS event names and payloads).
- `reference/events/core-pubsub` — Pub/Sub event reference (source of truth for async event names and payloads).

## Task Routing

- **Entity live sync** → Prefer optimistic mutation plus echo-suppressed streaming unless requirements prove otherwise. Load `references/optimistic-echo-suppression.md`.
- **Service async workflow** → Read Integration Patterns and Pub/Sub reference. Load `references/patterns.md`.
- **Frontend live UI** → Read Frontend principle and app docs. Load `references/client.md` for connection management.
- **Protocol selection** → Load `references/protocols.md`. Start from requirements (latency, directionality, multiplexing) and choose the simplest fit.
- **Backplane/replay work** → Read Real-Time and Reliability principles. Load `references/resiliency.md` and `references/system-design.md`.
- **Event contract design** → Load `references/contracts.md`. Check AsyncAPI specs for existing event conventions.
- **Scaling/fan-out** → Load `references/system-design.md`. Check current backplane architecture and capacity.

## Safety Gates

- Do not invent event names or payloads without checking AsyncAPI specs or source code. Events are contracts.
- Do not recommend CRDTs or event sourcing when simpler source-aware streaming satisfies the consistency requirements. Complexity has operational cost.
- Do not omit reconnection, replay, and duplicate handling from any real-time design. These are correctness requirements, not nice-to-haves.
- Do not design persistent connections without authentication, token refresh, and channel authorization.
- Run contract and docs health checks after event-doc changes.

## Hallucination Controls

Before presenting real-time guidance as factual:

- Check AsyncAPI specs for event names, payload schemas, and topic structures.
- Check source code for actual WebSocket/Pub/Sub implementations and connection patterns.
- Check the backplane architecture before making scaling or fan-out claims.
- Check existing client code for connection management patterns before proposing new ones.
- Label any recommendation based on general real-time knowledge (rather than Wordloop-specific patterns) as an inference.

## Output Expectations

- Protocol decisions include a justification based on actual requirements (latency, directionality, ordering, scale).
- Event designs include AsyncAPI contract changes, not just prose descriptions.
- Real-time features include explicit handling for reconnection, replay, echo suppression, and failure modes.
- Verification steps include specific commands for testing connection lifecycle and event delivery.
- Recommendations distinguish between Wordloop real-time conventions and general distributed systems practices.

## Antipatterns

Reject these patterns:

- **WebSocket by default** — Reaching for WebSockets when SSE would suffice. WebSockets add bidirectionality, connection management complexity, and proxy/load-balancer concerns. Use them when you need bidirectional communication.
- **Optimism without reconciliation** — Applying optimistic UI updates without server-confirmed reconciliation and rollback. Optimistic updates are a UX technique, not a data consistency strategy.
- **Fire-and-forget events** — Publishing events without delivery guarantees, retry logic, or dead-letter handling. Events that silently disappear are worse than events that fail loudly.
- **CRDT overkill** — Adopting conflict-free replicated data types for problems that don't have concurrent writes from multiple sources. CRDTs add conceptual and operational complexity; source-aware streaming handles most single-writer scenarios.
- **Reconnect and pray** — Reconnecting a WebSocket without replaying missed events from a known sequence position. The gap between disconnect and reconnect is where data gets lost.
- **Undocumented event contracts** — Adding events without AsyncAPI specs, treating event schemas as internal implementation details that consumers shouldn't depend on.
