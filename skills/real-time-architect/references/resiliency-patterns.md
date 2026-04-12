# Resiliency Patterns under Massive Scale

Persistent connection architectures fail in cascading, silent ways. Standard HTTP auto-scaling metrics fail because a dropped database doesn't result in HTTP 500s; it results in frozen sockets processing zero throughput.

## Graceful Draining & Node Lifecycle
When an API Gateway or Relay Pod receives a `SIGTERM` from the orchestrator (e.g., Kubernetes scaling down), abruptly dropping 50,000 WebSockets triggers a mass reconnection event that usually takes down the remaining fleet.

**The Solution:**
1. Use `preStop` hooks (delaying SIGTERM routing by 5-10 seconds) to ensure Load Balancer health checks deregister the pod *before* the application stops accepting connections.
2. The application enters "Draining State". It sends a Close Frame (`Code 1001: Going Away`) or SSE terminal flag to a staggered subset of its clients every few seconds.
3. This creates a smooth, phased reconnection across the remaining healthy fleet instead of a Thundering Herd.

## Thundering Herd Mitigation
If a core network failure occurs and 2 million clients drop instantly, they will all attempt to reconnect simultaneously.

*   **Client-Side:** Must implement Exponential Backoff with Jitter (Randomization Factor) when dialing back the handshake connection.
*   **Server-Side:** Use the **Generic Cell Rate Algorithm (GCRA)** instead of simple Token Buckets for distributed rate limiting. Evaluated via Lua scripts at the API Gateway or central Redis tier, GCRA ensures bursts of reconnection requests do not overwhelm the targeted Unicast infrastructure.

## Native Stream Backpressure
Do not rely on the application to manually buffer data if the client is slow.

*   **WebSockets (Legacy):** Requires manual monitoring of the socket's `bufferedAmount` property and implementing "drain" event listeners in the application code.
*   **WebTransport (Modern):** Built natively upon QUIC streams. If the receiver's buffer fills, the transport layer automatically signals backpressure to the publisher. The QUIC protocol pauses data transmission immediately, completely eliminating application-layer memory bloat and Out-of-Memory (OOM) risks on the server.

## Idempotency and Stream Safety
Because network jitter forces clients to replay recent events or blindly retry volatile state updates, real-time message streams must be strictly idempotent.

1. **Client-Generated Keys:** The client generates a unique `idempotencyKey` (UUID) locally alongside its Optimistic UI update.
2. **Act-If-Not-Exists:** The server attempts an atomic insert of this key into a centralized data store or database schema equipped with a `UNIQUE` constraint. 
3. **Conflict Replay:** If the insert fails, it is a duplicate retry. The server retrieves the original state result and pushes the cached confirmation back to the client, assuring confidence without duplicating side effects.
