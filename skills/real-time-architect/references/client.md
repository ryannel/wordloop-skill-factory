# Client-Side Patterns

Reference for client-side resilience, developer experience, and SDK design patterns. This covers everything that runs on the client — from offline queuing to type-safe event systems.

## Table of Contents

- [Offline Queue & Persistence](#offline-queue--persistence)
- [Optimistic UI & State Rollback](#optimistic-ui--state-rollback)
- [Reconnection Strategies](#reconnection-strategies)
- [Type-Safe Event Systems](#type-safe-event-systems)
- [SDK Generation](#sdk-generation)
- [Runtime Validation](#runtime-validation)

---

## Offline Queue & Persistence

Build clients that function during disconnection. Every action the user takes while offline should be captured, persisted, and drained when connectivity resumes.

```javascript
class OfflineQueue {
  constructor() {
    this.queue = [];
    this.db = null;
  }

  async init() {
    // Use IndexedDB for persistence across browser restarts
    this.db = await openDB("offline-queue", 1, {
      upgrade(db) {
        db.createObjectStore("pending", { keyPath: "idempotencyKey" });
      },
    });
  }

  async enqueue(action) {
    const entry = {
      ...action,
      idempotencyKey: action.idempotencyKey || crypto.randomUUID(),
      enqueuedAt: Date.now(),
    };

    this.queue.push(entry);
    await this.db.put("pending", entry);

    return entry.idempotencyKey;
  }

  async drain(socket) {
    const pending = await this.db.getAll("pending");

    for (const action of pending) {
      try {
        await socket.emitWithAck("action", action);
        await this.db.delete("pending", action.idempotencyKey);
      } catch (err) {
        // Server rejected — check if idempotencyKey conflict (duplicate)
        if (err.code === "DUPLICATE") {
          await this.db.delete("pending", action.idempotencyKey);
        }
        // else: keep in queue for next drain
      }
    }
  }
}
```

Key design decisions:
- **IndexedDB over localStorage**: IndexedDB supports structured data, larger storage limits, and transactional operations
- **Idempotency keys on every action**: Ensures safe drain-on-reconnect even if the client retries
- **FIFO ordering**: Actions are drained in the order they were enqueued to preserve causal ordering

---

## Optimistic UI & State Rollback

Apply changes to the UI immediately without waiting for server confirmation. Rollback if the server rejects:

```javascript
// 1. Apply optimistic update to UI immediately
function optimisticUpdate(action) {
  const rollback = applyToLocalState(action);
  pendingActions.set(action.idempotencyKey, rollback);
}

// 2. On server confirmation, finalize
socket.on("ack", ({ idempotencyKey }) => {
  pendingActions.delete(idempotencyKey);
});

// 3. On server rejection, rollback
socket.on("reject", ({ idempotencyKey, reason }) => {
  const rollback = pendingActions.get(idempotencyKey);
  if (rollback) {
    rollback(); // Revert the optimistic change
    pendingActions.delete(idempotencyKey);
    showError(reason);
  }
});
```

### React Integration Pattern

```typescript
function useOptimisticAction<T>(socket: Socket) {
  const [pending, setPending] = useState<Map<string, () => void>>(new Map());

  const dispatch = useCallback((action: Action) => {
    const key = crypto.randomUUID();
    const optimistic = { ...action, idempotencyKey: key };

    // Apply optimistically
    const rollback = applyToStore(optimistic);
    setPending((prev) => new Map(prev).set(key, rollback));

    // Send to server
    socket.emit("action", optimistic, (response: AckResponse) => {
      if (response.error) {
        // Rollback on failure
        const rb = pending.get(key);
        rb?.();
        setPending((prev) => {
          const next = new Map(prev);
          next.delete(key);
          return next;
        });
      } else {
        // Finalize on success
        setPending((prev) => {
          const next = new Map(prev);
          next.delete(key);
          return next;
        });
      }
    });
  }, [socket, pending]);

  return { dispatch, hasPending: pending.size > 0 };
}
```

---

## Reconnection Strategies

### Exponential Backoff with Jitter

Every reconnection strategy must include both exponential delay and randomized jitter to prevent thundering herd on server recovery:

```javascript
function calculateReconnectDelay(attempt, baseDelay = 1000, maxDelay = 30000) {
  // Exponential: 1s, 2s, 4s, 8s, 16s, 30s (capped)
  const exponential = Math.min(baseDelay * Math.pow(2, attempt), maxDelay);

  // Full jitter: random value between 0 and the exponential delay
  // This spreads reconnection attempts across the entire window
  const jitter = Math.random() * exponential;

  return jitter;
}

// Usage in client reconnection loop
let attempt = 0;

async function reconnect() {
  while (attempt < 10) {
    const delay = calculateReconnectDelay(attempt);
    console.log(`Reconnecting in ${Math.round(delay)}ms (attempt ${attempt + 1})`);
    await sleep(delay);

    try {
      await connect();
      attempt = 0; // Reset on success
      return;
    } catch {
      attempt++;
    }
  }
  console.error("Max reconnection attempts reached");
}
```

### Socket.IO Built-In Configuration

```javascript
const socket = io("wss://api.example.com", {
  reconnectionDelay: 1000,       // Base delay
  reconnectionDelayMax: 30000,   // Cap at 30s
  randomizationFactor: 0.5,      // 50% jitter: actual delay = delay * (1 ± 0.5)
  reconnectionAttempts: 10,
});

// Drain the offline queue after reconnection
socket.on("connect", async () => {
  if (offlineQueue.hasPending()) {
    await offlineQueue.drain(socket);
  }
});
```

---

## Type-Safe Event Systems

Prevent runtime bugs from `socket.on("mesage")` typos and incorrect payloads. Use TypeScript generic event maps to make events type-safe at compile time.

### Shared Event Map

```typescript
// shared/events.ts — shared between client and server
interface ServerToClientEvents {
  "document:updated": (update: DocumentUpdate) => void;
  "presence:changed": (event: PresenceEvent) => void;
  "message:received": (message: ChatMessage) => void;
  "auth-expired": (payload: { reason: string }) => void;
  "server-draining": (payload: { code: number; retryAfterMs: number }) => void;
  ack: (payload: { idempotencyKey: string }) => void;
  reject: (payload: { idempotencyKey: string; reason: string }) => void;
}

interface ClientToServerEvents {
  "document:edit": (
    edit: EditOperation,
    callback: (response: AckResponse) => void
  ) => void;
  "room:join": (roomId: string) => void;
  "room:leave": (roomId: string) => void;
  "message:send": (
    message: NewMessage,
    callback: (response: AckResponse) => void
  ) => void;
}

// Types
interface DocumentUpdate {
  documentId: string;
  version: number;
  operations: EditOperation[];
  updatedBy: { id: string; name: string };
}

interface PresenceEvent {
  userId: string;
  status: "online" | "idle" | "offline";
  lastSeen: string;
}

interface AckResponse {
  success: boolean;
  error?: string;
}
```

### Server Usage

```typescript
import { Server } from "socket.io";

const io = new Server<ClientToServerEvents, ServerToClientEvents>(httpServer);

io.on("connection", (socket) => {
  // TypeScript enforces event names and payload types
  socket.on("document:edit", (edit, callback) => {
    // 'edit' is typed as EditOperation
    // 'callback' is typed as (response: AckResponse) => void
    processEdit(edit);
    callback({ success: true }); // Type-checked
  });

  // Compile error: "documnet:updated" is not a valid event name
  // socket.emit("documnet:updated", { ... });

  // Correct: event name and payload are type-checked
  socket.emit("document:updated", {
    documentId: "doc_123",
    version: 42,
    operations: [],
    updatedBy: { id: "user_456", name: "Alice" },
  });
});
```

### Client Usage

```typescript
import { io, Socket } from "socket.io-client";

const socket: Socket<ServerToClientEvents, ClientToServerEvents> = io(
  "wss://api.example.com"
);

// Type-safe event listening
socket.on("document:updated", (update) => {
  // 'update' is automatically typed as DocumentUpdate
  console.log(`Version: ${update.version}`);
});

// Type-safe event emitting with acknowledgment
socket.emit("document:edit", editOperation, (response) => {
  // 'response' is typed as AckResponse
  if (response.error) {
    rollback();
  }
});
```

---

## SDK Generation

For teams exposing real-time APIs to external consumers, generate type-safe SDKs from AsyncAPI specs to ensure zero drift between the spec and the client code.

### AsyncAPI-Driven Generation

```bash
# Generate TypeScript client from the AsyncAPI spec
npx @asyncapi/cli generate fromTemplate asyncapi.yaml \
  @asyncapi/typescript-template -o ./sdk/generated/

# Or use the modelina tool for just the models
npx @asyncapi/modelina generate asyncapi.yaml \
  --language typescript --output ./sdk/models/
```

### Generated SDK Wrapper

```typescript
// sdk/client.ts — thin wrapper around generated types
import { ServerToClientEvents, ClientToServerEvents } from "./generated/events";
import { io, Socket } from "socket.io-client";

export class RealtimeClient {
  private socket: Socket<ServerToClientEvents, ClientToServerEvents>;

  constructor(url: string, token: string) {
    this.socket = io(url, {
      auth: { token },
      reconnectionDelay: 1000,
      reconnectionDelayMax: 30000,
      randomizationFactor: 0.5,
    });
  }

  // Typed subscription methods
  onDocumentUpdate(
    callback: (update: DocumentUpdate) => void
  ): () => void {
    this.socket.on("document:updated", callback);
    return () => this.socket.off("document:updated", callback);
  }

  // Typed action methods with async acknowledgment
  async editDocument(edit: EditOperation): Promise<AckResponse> {
    return new Promise((resolve) => {
      this.socket.emit("document:edit", edit, resolve);
    });
  }

  // Async iterator for stream consumption
  async *messages(conversationId: string): AsyncGenerator<ChatMessage> {
    this.socket.emit("room:join", conversationId);

    const messageQueue: ChatMessage[] = [];
    let resolve: (() => void) | null = null;

    this.socket.on("message:received", (msg) => {
      messageQueue.push(msg);
      resolve?.();
    });

    try {
      while (true) {
        if (messageQueue.length === 0) {
          await new Promise<void>((r) => (resolve = r));
        }
        yield messageQueue.shift()!;
      }
    } finally {
      this.socket.emit("room:leave", conversationId);
    }
  }
}
```

---

## Runtime Validation

TypeScript types exist only at compile time. For incoming events from the server (which is an external system), add runtime validation using Zod:

```typescript
import { z } from "zod";

// Define runtime schemas that mirror your TypeScript types
const DocumentUpdateSchema = z.object({
  documentId: z.string().uuid(),
  version: z.number().int().positive(),
  operations: z.array(
    z.object({
      type: z.enum(["INSERT", "DELETE", "UPDATE"]),
      path: z.string(),
      value: z.unknown(),
    })
  ),
  updatedBy: z.object({
    id: z.string(),
    name: z.string(),
  }),
});

type DocumentUpdate = z.infer<typeof DocumentUpdateSchema>;

// Validate incoming events before processing
socket.on("document:updated", (rawUpdate) => {
  const result = DocumentUpdateSchema.safeParse(rawUpdate);

  if (!result.success) {
    console.error("Invalid event from server:", result.error.format());
    // Report to observability — this indicates server-side drift
    telemetry.reportSchemaViolation("document:updated", result.error);
    return;
  }

  // result.data is now fully typed and validated
  applyDocumentUpdate(result.data);
});
```

### When to Use Runtime Validation

- **External APIs**: Always validate events from servers you don't control
- **Cross-team boundaries**: When producer and consumer teams deploy independently
- **Schema evolution**: During migration periods when old and new event shapes coexist
- **Production safety**: As a defense-in-depth layer even when you trust the producer
