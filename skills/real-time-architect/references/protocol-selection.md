# Protocol Selection & Transport Optimizations

Selecting the correct persistent transport layer requires matching the data directionality and latency constraints with the protocol's fundamental strengths. 

## The Protocol Decision Matrix

| Capability | Server-Sent Events (SSE) | WebTransport | WebSockets (WSS) |
| :--- | :--- | :--- | :--- |
| **Transport Base** | TCP or QUIC (HTTP/3) | QUIC (UDP Proxy) | TCP |
| **Directionality** | Unidirectional (Server $\rightarrow$ Client) | Bidirectional (Full-Duplex) | Bidirectional (Full-Duplex) |
| **Head-of-Line Blocking** | Stream-isolated if on HTTP/3 | None (Native QUIC multiplexing)| Yes (Protocol-wide blocking) |
| **Reconnection State** | Native (`Last-Event-ID`) | Native (Connection Migration)| Manual Application Logic |

### 1. Server-Sent Events (SSE)
**When to Use:** Live feeds, dashboards, system notifications, or streaming Large Language Model (LLM) tokens. If the client does not need to stream high-frequency data back to the server, do not pay the complexity tax of WebSockets.

**Critical Infrastructure Constraints:**
*   **Disable Proxy Buffering:** By default, reverse proxies (NGINX, AWS ALB) buffer HTTP responses. This will queue streaming SSE chunks until the buffer fills, breaking real-time delivery. You must explicitly configure the proxy to disable buffering for the specific route (e.g., returning the `X-Accel-Buffering: no` header).
*   **Heartbeats:** Keep-alive TCP timeouts are aggressive in modern cloud networks. The server must periodically send comment lines (`: heartbeat\n\n`) to keep the stream active.
*   **Native Resumption:** SSE natively handles drops. If the connection fails, the browser automatically reconnects, passing the `Last-Event-ID` header. The server backend must read this and replay missed events from the log.

### 2. WebTransport
**When to Use:** High-Frequency Trading (HFT) interfaces, Cloud Gaming, or collaborative cursor tracking.

*   **The QUIC Advantage:** Because WebTransport utilizes UDP-based QUIC, it allows multiple independent streams within one connection. If packet loss delays one stream, the others continue uninterrupted—eliminating Head-of-Line (HOL) blocking.
*   **Datagrams vs. Reliable Streams:** Use WebTransport *Reliable Streams* for critical state updates, and *Unreliable Datagrams* for ephemeral telemetry (e.g., mouse coordinates) where dropping a late frame is preferable to delaying the whole queue.
*   **Migration:** QUIC supports Transparent Connection Migration. If a mobile user switches from Wi-Fi to 5G, the Connection ID remains the same, surviving the IP change natively without renegotiating TLS.

### 3. Model Context Protocol (MCP) & AI Streams
When architecting for autonomous agents calling tools or consuming output, adopt **Streamable HTTP**. Instead of using a dual-channel setup (one REST endpoint for issuing commands and a separate SSE endpoint for listening), standard HTTP endpoints should dynamically establish an SSE stream within the single POST/GET request when real-time tool state or context streaming is required.
