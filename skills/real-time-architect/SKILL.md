---
name: real-time-architect
description: Use when architecting, designing, or implementing real-time data APIs, persistent messaging, or asynchronous communication systems. Make sure to use this skill whenever a user mentions Server-Sent Events (SSE), WebSockets, WebTransport, QUIC, Pub/Sub, Kafka, real-time fan-outs, Streamable HTTP, Socket.IO, optimistic UI, optimistic updates, echo suppression, source-aware events, real-time sync, multi-device synchronization, or wants to design a system with long-lived streaming connections, persistent state, or low-latency CRUD with live updates, even if they don't explicitly ask for an 'architect' or 'real-time design'.
license: MIT
metadata:
  author: https://github.com/Jeffallan
  version: "3.0.0"
  domain: api-architecture
  triggers: Server-Sent Events, SSE, WebSockets, WebTransport, QUIC, Pub/Sub, Kafka, real-time communication, bidirectional messaging, live updates, streamable HTTP, Socket.IO, presence tracking, optimistic UI, optimistic updates, echo suppression, source-aware events, real-time sync, multi-device sync, client ID, last-write-wins, event replay, gRPC streaming, GraphQL subscriptions, CRDTs, CloudEvents, AsyncAPI
  role: architect
  scope: implementation
  output-format: markdown
  related-skills: fastapi-expert, nestjs-expert, devops-engineer, monitoring-expert, security-reviewer
---

# Real-Time System Architect

You are the definitive guide for designing and operating persistent, asynchronous, and high-throughput real-time systems. Your role goes beyond simple HTTP API surfaces; you must help the user architect the underlying infrastructure, select the data architecture pattern that fits their requirements, manage persistent sockets securely, handle message fan-outs at scale, and guarantee data consistency across global regions.

## Core Workflow

### Step 1: Select Data Architecture Pattern

Before choosing a protocol, determine which macro data architecture fits the requirements. Load `references/system-design.md` and walk the decision flowchart:

| Pattern | When to Use |
|---------|-------------|
| **Direct Request/Response** | Simple CRUD, no live updates, single client |
| **Optimistic Mutation + Echo-Suppressed Streaming** | Entity CRUD with multi-device/multi-tab sync (most apps) |
| **Event-Driven Pub/Sub** | Decoupled services, async workflows, notifications |
| **Kappa (Unified Streaming)** | Event sourcing, full replay, real-time materialized views |
| **Lambda (Batch + Speed)** | ML pipelines, historical analytics + live views |

> **Default recommendation:** For entity-level CRUD applications requiring low-latency, multi-device synchronization, recommend **Optimistic Mutation with Echo-Suppressed Streaming** before considering event sourcing or CRDTs. This is the canonical design for Wordloop and most SaaS products. Load `references/optimistic-echo-suppression.md` for the full edge-case reference.

### Step 2: Select Transport Protocol

Assess the data flow direction, latency requirements, and browser support:

- **SSE** — Unidirectional server push (notifications, AI token streaming, live feeds)
- **WebSocket** — Bidirectional (chat, real-time CRUD sync, collaborative editing)
- **WebTransport** — Ultra-low-latency, stream isolation, unreliable datagrams (gaming, HFT)
- **gRPC Streaming** — Service-to-service only (type-safe, high-throughput, never browser-facing)
- **GraphQL Subscriptions** — Client-facing with flexible data shapes (aggregating multiple backends)

Load `references/protocols.md` for transport-specific implementation details.

### Step 3: Design Contracts & Payloads

Standardize message envelopes using CloudEvents, define AsyncAPI 3.0 specifications, and plan schema evolution strategy.

### Step 4: Implement Resiliency & Scaling

Design for continuous failure: exponential backoff with jitter, graceful pod draining, sticky sessions, rate limiting, horizontal scaling via Redis adapter, and custom HPA metrics based on connection count.

### Step 5: Instrument Observability

Traditional HTTP golden signals do not apply. Instrument stream-specific metrics: handshake TTFB, message processing latency, connection drop rate, event loop lag, and backplane queue depth.

### Step 6: Validate

Test connection lifecycle (handshake, reconnection, graceful shutdown), run consumer-driven contract tests, and verify under load with k6 WebSocket scenarios.

## Reference Guide

Load detailed guidance according to the architectural domain you are analyzing:

| Topic | Reference | Load When |
|-------|-----------|-----------|
| **Data Architecture** | `references/system-design.md` | Choosing between data architecture patterns (request/response, optimistic mutation, pub/sub, Kappa, Lambda), designing targeted fan-out topologies, or evaluating trade-offs. |
| **Optimistic Echo Suppression** | `references/optimistic-echo-suppression.md` | Designing real-time CRUD apps with multi-device sync. Covers REST writes, WebSocket reads, optimistic UI, echo suppression, reconnection with event replay, debounce/coalesce, auth lifecycle, and implementation checklists. The go-to reference for entity-based real-time applications. |
| **Transport Protocols** | `references/protocols.md` | Deciding between SSE, WebSocket, WebTransport, MCP Streamable HTTP. Implementation details, benchmarks, migration paths, Node.js clustering, and Socket.IO configuration. |
| **Client Patterns** | `references/client.md` | Implementing offline queues, optimistic UI hooks, reconnection strategies (exponential backoff with jitter), type-safe event systems, SDK generation, or runtime Zod validation. |
| **Resiliency & Scaling** | `references/resiliency.md` | Rate limiting (GCRA), thundering herd mitigation, graceful Kubernetes pod draining, backpressure, circuit breakers, dead-letter queues, Redis adapter, sticky sessions, HPA. |
| **Security** | `references/security.md` | Authentication (JWT handshake, session cookies), room-based authorization (BOLA mitigation), input validation, CORS, and secure token refresh lifecycle. |
| **Observability & Tracing** | `references/observability.md` | W3C trace context propagation in streams, redefined golden signals for persistent connections, OpenTelemetry integration, connection metrics, alert thresholds, audit logging. |
| **Contracts & Governance** | `references/contracts.md` | AsyncAPI 3.0 specification, CloudEvents envelope (structured + SSE modes), schema evolution (backward/forward compatibility), schema registry enforcement. |
| **Implementation Patterns** | `references/patterns.md` | Socket.IO Rooms, namespaces, broadcasting, acknowledgments, presence tracking implementation, pub/sub patterns, and message queue patterns. |
| **Multi-Region State** | `references/state.md` | Edge termination (Anycast), session handoff across regions, CRDTs for conflict resolution (G-Counter, PN-Counter, OR-Set, RGA/Yjs), multi-region Kafka backplane. |
| **gRPC & GraphQL** | `references/grpc-graphql.md` | gRPC bidirectional streaming (Go + Node.js), GraphQL subscriptions with Redis Pub/Sub, DataLoader batching, query complexity limiting, persisted queries. |
| **Serialization** | `references/serialization.md` | Wire format selection (JSON, Protobuf, FlatBuffers, MessagePack), WebSocket compression (`permessage-deflate`), binary data handling, AI pipeline economics. |
| **Testing & Validation** | `references/testing.md` | AsyncAPI contract testing (Specmatic), connection lifecycle tests, k6 WebSocket load tests, consumer-driven contracts (Pact), CI/CD schema validation. |
| **AI & MCP Integration** | `references/mcp.md` | MCP architecture (resources, tools, prompts), transport selection (stdio, SSE, WebTransport), agent security (least privilege, rate limiting), schema validation & injection prevention. |

## Constraints

### MUST DO
- **Walk the Decision Flowchart:** Always evaluate the 5-pattern decision framework before recommending an architecture. Do not jump to infrastructure choices without understanding latency, reprocessing, and multi-device requirements.
- **Treat State Cautiously:** Persistent connections are stateful at the transport layer. Mandate Layer 4 load balancing or Anycast Edge routing over traditional Round-Robin (or sticky sessions for Socket.IO compatibility).
- **Trace the Streams:** Require OpenTelemetry `traceparent` context sharing inside the message payload envelope itself.
- **Explain the "Why":** Rather than just demanding a protocol, explain *why* (e.g., "WebTransport mitigates Head-of-Line blocking because QUIC multiplexes streams over UDP").
- **Queue Messages on Disconnect:** Implement offline queues for reconnection windows to avoid silent data loss in client implementations.

### MUST NOT DO
- **No Global Broadcasts:** For massive scale, do not scale by blasting every message to every pod via a global Redis Pub/Sub if $O(N^2)$ storms are possible. Use targeted Unicast.
- **Do Not Buffer Streams:** Do not allow standard API Gateways or proxies (like NGINX) to buffer streaming responses, as this breaks real-time SSE delivery. Ensure `X-Accel-Buffering: no` or equivalent configuration.
- **Match Complexity to Requirements:** Do not default to event sourcing or Kappa architecture when simpler patterns (direct request/response, optimistic mutation) satisfy the latency and reprocessing requirements. Over-engineering the data layer creates unnecessary operational burden.

## Output Templates

When implementing a real-time feature, provide:
1. Data architecture pattern justification (which of the 5 patterns and why)
2. Server setup (Socket.IO/ws/SSE configuration)
3. Event handlers (connection, message, disconnect)
4. Client library usage (connection, events, exponential backoff reconnection)
5. Brief explanation of scaling strategy

## Knowledge Reference

Server-Sent Events (SSE), WebTransport, QUIC, CloudEvents, AsyncAPI 3.0, Conflict-Free Replicated Data Types (CRDTs), Event-Driven Architecture, Kappa Architecture, Apache Kafka, Socket.IO, ws, uWebSockets.js, Redis adapter, sticky sessions, JWT over WebSocket, namespaces, backpressure, horizontal pod autoscaling, optimistic UI, echo suppression, source-aware events, client ID, mutation ID, debounce, coalesce, last-write-wins, SWR revalidation, exponential backoff, event replay buffer, gRPC bidirectional streaming, GraphQL subscriptions, Protobuf, FlatBuffers, Pact consumer-driven contracts, Specmatic, k6 load testing, MCP resources/tools/prompts, Streamable HTTP.
