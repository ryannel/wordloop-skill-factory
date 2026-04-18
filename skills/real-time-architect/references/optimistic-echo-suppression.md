# Optimistic Mutation with Echo-Suppressed Streaming

A complete architectural pattern for applications that need low-latency CRUD operations with real-time multi-device synchronization. This pattern separates writes (REST) from reads (WebSocket) and uses source-aware events to prevent echo on the originating client.

This pattern is used by Linear, Figma, Notion, and other modern real-time applications. It sits between simple request/response CRUD and full event sourcing — delivering real-time push updates without the operational complexity of an immutable log.

## Table of Contents

- [When to Use This Pattern](#when-to-use-this-pattern)
- [Architecture Overview](#architecture-overview)
- [The Five-Step Data Loop](#the-five-step-data-loop)
- [Client Identity](#client-identity)
- [Event Scoping](#event-scoping)
- [Initial State Hydration](#initial-state-hydration)
- [Edge Cases](#edge-cases)
- [Implementation Checklist](#implementation-checklist)

---

## When to Use This Pattern

**Use when:**
- The application has entity-level CRUD operations (notes, tasks, items, messages).
- Multiple clients (devices, tabs) for the same user must stay in sync in real time.
- Users expect instant feedback — perceived zero-latency mutations.
- The data model is entity-based (not collaborative text editing).

**Do not use when:**
- The application requires collaborative editing with character-level conflict resolution — use CRDTs or Operational Transform.
- The data is append-only and requires full replay — use Kappa / event sourcing.
- There is no multi-device or multi-user requirement — simple REST is sufficient.
- The application is read-only with no user-initiated mutations — use SSE or WebSocket streaming alone.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                           CLIENT                                │
│                                                                 │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────────────┐  │
│  │ UI State    │◄───│ SWR / Cache │◄───│ WebSocket Listener  │  │
│  │ (Optimistic)│    │             │    │ (Echo Suppression)  │  │
│  └──────┬──────┘    └──────▲──────┘    └──────────▲──────────┘  │
│         │                  │                      │             │
│         ▼                  │                      │             │
│  ┌─────────────┐           │                      │             │
│  │ REST Client │───────────┘                      │             │
│  │ (Mutations) │                                  │             │
│  └──────┬──────┘                                  │             │
└─────────┼─────────────────────────────────────────┼─────────────┘
          │ REST (CUD)                              │ WebSocket (Read)
          ▼                                         │
┌─────────────────────────────────────────────────────────────────┐
│                           SERVER                                │
│                                                                 │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────────────┐  │
│  │ REST API    │───▶│ Database    │───▶│ WebSocket Hub       │  │
│  │ (Handlers)  │    │ (Write)     │    │ (Event Broadcast)   │  │
│  └─────────────┘    └─────────────┘    └─────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

**Write path:** Client → REST API → Database → WebSocket Hub → All Clients
**Read path:** WebSocket Hub → Client (complete entity payloads, not diffs)

---

## The Five-Step Data Loop

Every user-initiated mutation follows this exact sequence:

### Step 1 — Optimistic Update (Client)

The client applies the change to local state immediately, before the network request fires.

1. **Apply change to UI.** The user sees the result in under 16ms.
2. **Store rollback snapshot.** If the server rejects, revert to this state.
3. **Show pending indicator.** Subtle visual cue (reduced opacity, spinner) until confirmed.
4. **Generate temporary ID** (for creates). A UUID prefixed with `temp_` so the new entity can appear in the UI before the server assigns the real ID.

### Step 2 — REST Mutation (Client → Server)

Send the mutation to the REST endpoint with two critical headers:

```http
POST /api/v1/notes HTTP/1.1
Authorization: Bearer <jwt>
X-Client-Id: abc-123
Content-Type: application/json

{
  "meetingId": "mtg_01J...",
  "content": "Follow up with the design team"
}
```

| Header | Purpose |
|---|---|
| `Authorization` | User identity (JWT). Determines **who** is performing the action. |
| `X-Client-Id` | Client instance identity. Determines **which device/tab** initiated the action. Used exclusively for echo suppression. |

The REST response returns the **complete server-authoritative entity** (server-assigned `id`, timestamps, `version`). The client replaces its temporary optimistic state with this confirmed entity.

For **create** operations, also send an `Idempotency-Key` header to prevent duplicate creation on retry:

```http
POST /api/v1/notes HTTP/1.1
Idempotency-Key: idem_7f3a9c...
X-Client-Id: abc-123
```

### Step 3 — Event Broadcast (Server → WebSocket Hub)

After the database write succeeds, the server publishes a WebSocket event to all connected clients within the event's scope. The event carries the **complete entity payload** (not a diff) tagged with the `sourceClientId`:

```json
{
  "specversion": "1.0",
  "type": "note.created",
  "source": "wordloop-core",
  "id": "evt_01J...",
  "data": {
    "id": "note_01J...",
    "meetingId": "mtg_01J...",
    "content": "Follow up with the design team",
    "createdAt": "2026-04-17T20:00:00Z",
    "updatedAt": "2026-04-17T20:00:00Z",
    "version": 1
  },
  "sourceClientId": "abc-123"
}
```

**Why complete payloads?** Receiving clients can replace their local copy of the entity directly — no patch operations, no change log, no CRDT merge. The trade-off is larger payloads, which is acceptable for typical entity sizes (under 10KB).

### Step 4 — Echo Suppression (Originating Client)

The originating client receives the WebSocket event and compares `sourceClientId` against its own ID:

```
Incoming: sourceClientId = "abc-123"
My ID:    clientId        = "abc-123"
→ Match. Discard (UI already reflects from optimistic update).
```

Without echo suppression, the originator renders the change twice — once from optimistic update, once from WebSocket — causing visual flicker and duplicate list entries.

### Step 5 — Cross-Device Sync (Other Clients)

Other clients for the same user receive the same event. Their `clientId` does NOT match `sourceClientId`, so they apply the payload directly to local state — **no REST call needed**.

```
Incoming: sourceClientId = "abc-123"
My ID:    clientId        = "def-456"
→ No match. Apply entity to local state. UI updates in real time.
```

---

## Client Identity

### Client ID Design

A `clientId` is a UUID generated **per browser tab** when the application initializes.

| Property | Value |
|---|---|
| **Scope** | One per browser tab / app instance |
| **Lifetime** | Created on tab open, discarded on tab close |
| **Persistence** | `sessionStorage` (survives refresh within same tab, not across tabs) |
| **Format** | UUIDv4 |

**Why per-tab, not per-session?** If shared across tabs, a mutation from Tab A would suppress the WebSocket event in Tab B — Tab B would never render the change.

### Client ID vs Mutation ID

Some architectures use a unique `mutationId` per operation. The trade-off:

| Approach | Pros | Cons |
|---|---|---|
| **Client ID** | Simpler — one header, single string comparison for suppression. | Cannot distinguish between two of your own rapid mutations on the same entity. |
| **Mutation ID** | Precisely reconciles individual operations. | Requires pending-mutation queue on client and mutation-ID propagation through server. |

**When to switch to mutation IDs:** When introducing collaborative editing where individual keystrokes must be tracked. For standard entity CRUD, client IDs are sufficient.

---

## Event Scoping

Events are scoped to the relevant audience, not broadcast globally:

| Scope | Events Delivered To | Example |
|---|---|---|
| **User** | All clients authenticated as that user | `note.created`, `task.updated`, `meeting.deleted` |
| **Resource** | All clients viewing that specific resource | `transcript.segment.produced`, `insight.generated` |

The WebSocket hub maintains a mapping of `userId → [connections...]` and a subscription registry of `resourceId → [connections...]`.

---

## Initial State Hydration

When a client first loads, the WebSocket is not yet connected. The hydration sequence is:

1. **REST fetch** — Load current state via REST (e.g., `GET /meetings/:id`).
2. **Render UI** — Display initial state.
3. **Connect WebSocket** — Include JWT and `clientId` in handshake.
4. **Bridge the gap** — Include a `since` timestamp (from the REST response's latest `updatedAt`) in the WebSocket handshake. The server replays any events that occurred between the REST response and WebSocket connection.
5. **Resume live stream** — Apply WebSocket events to UI as they arrive.

Without step 4, events during the brief window between REST response and WebSocket connection would be lost.

---

## Edge Cases

### Mutation Failure and Rollback

The rollback strategy depends on the error category:

| Error Category | HTTP Status | Retry? | Behavior |
|---|---|---|---|
| **Validation / Client Error** | 400, 409, 422 | No | Roll back immediately. Surface error. |
| **Authentication Error** | 401, 403 | No | Roll back. Redirect to login or refresh token. |
| **Not Found** | 404 | No | Roll back. Entity deleted by another client. |
| **Server Error** | 500, 502, 503 | Yes | Retry with exponential backoff + jitter (1s, 2s, 4s, max 3 retries). Roll back after exhaustion. |
| **Network Timeout** | — | Yes | Retry once. Roll back if still fails. Surface ambiguous error. |

### Optimistic ID Reconciliation

For entity creation, the client generates a temporary ID (`temp_xxx`). When the REST response returns the server-assigned ID, the client replaces the temporary ID everywhere in local state. The subsequent WebSocket echo is suppressed by `clientId` matching.

### WebSocket Event Arrives Before REST Response

Safe — echo suppression discards the event regardless of timing. The REST response confirms the mutation independently.

### Concurrent Mutations (Last-Write-Wins)

If two devices edit the same entity simultaneously, the last write to reach the database wins. Both REST calls succeed, both produce WebSocket events. Each client receives the other's event and replaces local state.

**When this is insufficient:** If collaborative editing is needed (simultaneous text editing within a document), shift to CRDTs or Operational Transform. Last-write-wins is appropriate for entity-level operations where conflicts are rare.

### Delete Race Condition

If Client A deletes an entity while Client B is editing it:

1. Client A's `DELETE` succeeds. Server publishes `entity.deleted`.
2. Client B receives `entity.deleted` and removes the entity from UI — even if Client B has unsaved optimistic changes.
3. If Client B's `PATCH` arrives after the delete, server returns `404`. Client B rolls back.

**The delete always wins.** The client must close the editor and surface a notification.

### Stale Event Ordering

Under network jitter, events for the same entity can arrive out of order. Each entity carries a `version` field (monotonically incrementing integer):

```
Local state:    { id: "note_01J...", version: 3 }
WS event:       { id: "note_01J...", version: 2 }
→ Event version < local version. Discard as stale.
```

Never apply an event whose version is ≤ the local version for the same entity.

### Reconnection and Missed Events

When the WebSocket drops, events during the disconnect window are lost. The client must:

1. Track the `id` of the last received event.
2. On reconnect, send `lastEventId` in the handshake.
3. Server replays all events after that ID from a short-lived event buffer.
4. If disconnected longer than the buffer window → full state re-fetch via REST.

**Reconnection backoff:** Exponential backoff with ±30% jitter to prevent thundering herds:

| Attempt | Base Delay | With Jitter |
|---|---|---|
| 1 | 1s | 0.7–1.3s |
| 2 | 2s | 1.4–2.6s |
| 3 | 4s | 2.8–5.2s |
| 4 | 8s | 5.6–10.4s |
| 5+ | 16s (cap) | 11.2–20.8s |

### Rapid Mutations (Debounce + Coalesce)

For rapid edits on the same entity (typing, sliders):

1. **Debounce REST call** (300–500ms pause). Optimistic UI still applies every keystroke immediately.
2. **Coalesce intermediate states.** Only the final state is sent to the server.
3. **Cancel stale in-flight requests** via `AbortController`.

### Tab Focus Revalidation

When a tab regains focus after backgrounding, the WebSocket may have silently disconnected. Treat tab-focus as a trigger to:

1. Check WebSocket health. If dead, reconnect with `lastEventId`.
2. Revalidate stale queries (SWR `revalidateOnFocus` or equivalent).

### WebSocket Authentication Lifecycle

JWTs expire. The connection must handle token refresh and session revocation:

**Proactive refresh:** Client monitors JWT expiry → refreshes token → sends `auth.refresh` message over existing WebSocket → server re-associates. No reconnection needed.

**Server-initiated revocation:** User logs out / admin revokes → server sends `session.revoked` to all connections → clients clear state and redirect to login.

**Reactive expiry:** Token expires without refresh → server sends close frame with custom code `4401` → client refreshes token and reconnects.

### Partial Server Failure

DB write succeeds, WebSocket broadcast fails:

- **Originator:** Receives REST response. Knows mutation succeeded.
- **Other clients:** Miss the event. Eventually consistent — corrected on next REST fetch (page nav, tab focus) or WebSocket reconnect with replay.

### Cache Reconciliation on Settled

After every mutation (success or failure), revalidate the affected cache key:

- **On success:** Update cache with REST response. Trigger background revalidation to catch concurrent edits.
- **On error:** Rollback restores snapshot. Revalidation fetches current server state.

This self-healing pattern ensures the cache is always consistent, even if echo suppression or version logic has a subtle bug.

---

## Implementation Checklist

When implementing this pattern, verify every item:

### Server-Side
- [ ] REST endpoints return complete entity (including `id`, `version`, `updatedAt`)
- [ ] `X-Client-Id` header extracted and propagated to WebSocket hub
- [ ] `Idempotency-Key` header honored for POST operations (short-lived cache)
- [ ] WebSocket event payload includes full entity and `sourceClientId`
- [ ] Events scoped to user/resource (not global broadcast)
- [ ] Event buffer maintained for reconnection replay (`lastEventId`)
- [ ] JWT validated on WebSocket handshake and `auth.refresh` messages
- [ ] `session.revoked` event published on logout to all user connections
- [ ] Custom close code `4401` sent on token expiry

### Client-Side
- [ ] `clientId` generated per-tab, stored in `sessionStorage`
- [ ] Optimistic update applied before REST call, with rollback snapshot stored
- [ ] Pending indicator shown on optimistic entities
- [ ] Temporary IDs (`temp_xxx`) generated for creates, reconciled on REST response
- [ ] Echo suppression: `sourceClientId === myClientId` → discard
- [ ] Version comparison: incoming `version ≤ local version` → discard
- [ ] Error classification: 4xx no retry, 5xx + timeout retry with backoff
- [ ] Rapid edits debounced (300–500ms), in-flight cancelled via `AbortController`
- [ ] WebSocket reconnection with exponential backoff + jitter
- [ ] `lastEventId` tracked and sent on reconnection handshake
- [ ] Tab focus triggers WebSocket health check + SWR revalidation
- [ ] JWT expiry monitored, `auth.refresh` sent proactively
- [ ] `session.revoked` event handler clears state and redirects to login
- [ ] `onSettled` revalidation after every mutation (success and failure)
