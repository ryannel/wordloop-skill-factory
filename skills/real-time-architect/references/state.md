# Multi-Region State & Conflict Resolution

Deep reference for operating real-time systems across multiple regions with active-active deployments. Traditional locking mechanisms are too slow for global real-time systems — this reference covers eventually consistent approaches.

## Table of Contents

- [Edge Termination & Anycast](#edge-termination--anycast)
- [Session Handoff](#session-handoff)
- [CRDTs for Conflict Resolution](#crdts-for-conflict-resolution)
- [Multi-Region Backplane](#multi-region-backplane)

---

## Edge Termination & Anycast

### Transport Termination at the Edge

Terminate TLS and transport at the physical edge closest to the user using Anycast IPs. This shields the connection from the volatile public internet — internally, the connection routes over the provider's private backbone.

```
User (Tokyo) → Anycast IP → Edge POP (Tokyo) → Private Backbone → Origin (US-West)
```

Benefits:
- Reduced handshake latency (TLS terminates at the nearest POP)
- Reduced exposure to public internet jitter and packet loss
- Connection appears local to the client while data routes globally

### Anycast Failover

If a data center fails, Anycast automatically routes clients to the next closest region. The client does not need to discover a new endpoint — the same IP address now resolves to a different POP.

```javascript
// Client-side: same URL, Anycast handles routing
const socket = io("wss://global.example.com", {
  auth: { token: getAuthToken() },
  // No region-specific configuration needed
});
```

---

## Session Handoff

When a client fails over to a new region (data center failure, physical relocation), the new server must reconstruct the session state:

### Resume Protocol

1. Client reconnects to a new region (Anycast reroute)
2. Client sends its `Last-Event-ID` (SSE) or `Producer-Epoch` + `Producer-Seq` (WebTransport)
3. New region queries the durable backplane (Kafka) for missed events since that checkpoint
4. New region replays missed events to the client
5. Normal streaming resumes

```javascript
io.on("connection", async (socket) => {
  const lastEventId = socket.handshake.query.lastEventId;
  const userId = socket.data.user.id;

  if (lastEventId) {
    // Fetch missed events from the durable backplane
    const missedEvents = await kafka.consume({
      topic: `user-events-${userId}`,
      fromOffset: lastEventId,
    });

    // Replay to the client
    for (const event of missedEvents) {
      socket.emit("replay", event);
    }

    socket.emit("replay-complete", {
      resumedFrom: lastEventId,
      eventsReplayed: missedEvents.length,
    });
  }

  // Normal event streaming continues
  subscribeToLiveEvents(socket, userId);
});
```

### State Consistency During Handoff

- The durable backplane (Kafka) retains events for a configurable retention window (e.g., 7 days)
- Event ordering is guaranteed within a Kafka partition (partition by user/room ID)
- The client must handle duplicate events gracefully using the `idempotencyKey` in the CloudEvents envelope

---

## CRDTs for Conflict Resolution

When users in different regions modify the same data simultaneously, traditional distributed locks are too slow for real-time global systems. CRDTs (Conflict-Free Replicated Data Types) are mathematical data structures designed so that concurrent updates can be applied independently in any order. Because operations are strictly commutative, replicas in different regions will eventually converge to the exact same state without any coordination.

### Why Not Locks?

| Approach | Latency | Availability | Consistency |
|----------|---------|-------------|-------------|
| Distributed Lock (Raft/Paxos) | 50-200ms (cross-region RTT) | Blocks during lock contention | Strong |
| Last-Write-Wins (LWW) | 0ms (local) | Always available | Data loss on conflict |
| **CRDTs** | 0ms (local) | Always available | Eventual (mathematically guaranteed) |

For real-time collaboration, CRDTs provide the correct balance: instant local responsiveness with guaranteed eventual consistency.

### Common CRDT Types

| Type | Use Case | Behavior |
|------|----------|----------|
| **G-Counter** | Like counts, view counts | Only increments; merges by taking max per replica |
| **PN-Counter** | Upvotes/downvotes | Separate increment/decrement counters; merges by taking max of each |
| **LWW-Register** | User profile fields | Last-Writer-Wins by timestamp; no conflict possible |
| **OR-Set** (Observed-Remove Set) | Tags, member lists | Add/remove with unique IDs; removes only affect observed adds |
| **RGA** (Replicated Growable Array) | Collaborative text editing | Ordered list with unique position IDs; concurrent inserts get deterministic positions |

### Implementation Pattern

```javascript
// G-Counter CRDT: each replica tracks its own counter
class GCounter {
  constructor(replicaId) {
    this.replicaId = replicaId;
    this.counters = {};
  }

  increment(amount = 1) {
    this.counters[this.replicaId] =
      (this.counters[this.replicaId] || 0) + amount;
  }

  // Merge takes the max of each replica's counter
  merge(other) {
    for (const [replica, count] of Object.entries(other.counters)) {
      this.counters[replica] = Math.max(
        this.counters[replica] || 0,
        count
      );
    }
  }

  value() {
    return Object.values(this.counters).reduce((sum, c) => sum + c, 0);
  }
}

// Usage across regions
const euCounter = new GCounter("eu-west-1");
const usCounter = new GCounter("us-east-1");

euCounter.increment(5);   // EU user clicks 5 times
usCounter.increment(3);   // US user clicks 3 times

// After replication
euCounter.merge(usCounter);
usCounter.merge(euCounter);

// Both converge to 8
console.log(euCounter.value()); // 8
console.log(usCounter.value()); // 8
```

### CRDT in Collaborative Editing

For document editing, use an RGA (Replicated Growable Array) or a tree-based CRDT like Yjs or Automerge:

```javascript
import * as Y from "yjs";

// Each region maintains its own Yjs document
const ydoc = new Y.Doc();
const ytext = ydoc.getText("content");

// Local edits apply instantly
ytext.insert(0, "Hello from EU");

// Sync state to other regions via the backplane
ydoc.on("update", (update) => {
  backplane.publish("doc-sync", {
    documentId,
    update: Buffer.from(update).toString("base64"),
    region: process.env.REGION,
  });
});

// Apply remote updates — order doesn't matter
backplane.subscribe("doc-sync", (msg) => {
  if (msg.region !== process.env.REGION) {
    const update = Buffer.from(msg.update, "base64");
    Y.applyUpdate(ydoc, update);
    // Document converges automatically
  }
});
```

### Convergence Monitoring

Track convergence lag between regions:

```javascript
// Each region periodically broadcasts its version vector
const versionVector = meter.createHistogram("crdt.convergence_lag_ms", {
  description: "Time for CRDT state to converge across regions",
  unit: "ms",
});

// Measure: time between a local write and the remote replica confirming receipt
function trackConvergence(eventId, localTimestamp) {
  backplane.once(`ack:${eventId}`, (remoteTimestamp) => {
    versionVector.record(remoteTimestamp - localTimestamp, {
      from: process.env.REGION,
    });
  });
}
```

---

## Multi-Region Backplane

### Topology

```
┌──────────────┐          ┌──────────────┐
│  EU Region   │          │  US Region   │
│              │          │              │
│ ┌──────────┐ │  Kafka   │ ┌──────────┐ │
│ │ Pods     │◄├──MirrorMaker──►│ Pods     │ │
│ └──────────┘ │          │ └──────────┘ │
│ ┌──────────┐ │          │ ┌──────────┐ │
│ │ Kafka    │◄├──────────►│ Kafka    │ │
│ │ Cluster  │ │          │ │ Cluster  │ │
│ └──────────┘ │          │ └──────────┘ │
└──────────────┘          └──────────────┘
```

### Cross-Region Replication

- **Kafka MirrorMaker 2** replicates topics between regional clusters
- Partition by user/room ID to maintain per-entity ordering
- Use geo-aware topic naming: `events.us-east-1`, `events.eu-west-1`
- Consumer groups in each region read from both local and mirrored topics

### Conflict-Free Event Routing

```javascript
// Each region publishes events with region metadata
async function publishEvent(event) {
  await kafka.produce({
    topic: `events.${process.env.REGION}`,
    key: event.data.entityId,  // Partition by entity
    value: JSON.stringify({
      ...event,
      metadata: {
        ...event.metadata,
        region: process.env.REGION,
        wallClockTime: Date.now(),
        logicalClock: incrementLogicalClock(),
      },
    }),
  });
}
```

### Ordering Guarantees

| Scope | Guarantee | Mechanism |
|-------|-----------|-----------|
| Within a partition | Total order | Kafka offset |
| Within a region | Causal order | Logical clocks |
| Across regions | Eventual convergence | CRDTs + MirrorMaker |
