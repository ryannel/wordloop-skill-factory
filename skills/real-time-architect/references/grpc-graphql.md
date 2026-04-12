# gRPC Streaming & GraphQL Subscriptions

Production real-time architectures are polyglot — they use different protocols at different boundaries. This reference covers the internal (gRPC) and client-facing (GraphQL) streaming layers that complement the edge delivery protocols (SSE, WebSocket, WebTransport) covered in `protocols.md`.

## Table of Contents

- [Architecture: The Polyglot Gateway](#architecture-the-polyglot-gateway)
- [gRPC Bidirectional Streaming](#grpc-bidirectional-streaming)
- [GraphQL Subscriptions](#graphql-subscriptions)
- [Protocol Decision Matrix](#protocol-decision-matrix)

---

## Architecture: The Polyglot Gateway

Most scalable systems in 2026 do not rely on a single protocol. The standard layered architecture is:

```
┌──────────────────────────────────┐
│  Clients (Web, Mobile, Agents)   │
└──────────────┬───────────────────┘
               │  GraphQL / SSE / WebSocket / WebTransport
┌──────────────▼───────────────────┐
│   BFF / API Gateway              │  Protocol translation, response shaping
│   (GraphQL layer aggregates      │  Query complexity limiting, auth
│    internal gRPC services)       │
└──────────────┬───────────────────┘
               │  gRPC (Protobuf, HTTP/2)
┌──────────────▼───────────────────┐
│   Internal Microservices         │  High-throughput, type-safe
│   (Order Service, User Service,  │  Bidirectional streaming
│    Notification Service)         │
└──────────────┬───────────────────┘
               │
┌──────────────▼───────────────────┐
│   Backplane (Kafka/NATS/Redis)   │  Cross-service event routing
└──────────────────────────────────┘
```

### Backend-for-Frontend (BFF) Pattern

The BFF sits between clients and internal services:

- **Inbound:** Accepts GraphQL queries/subscriptions or WebSocket/SSE connections from clients
- **Outbound:** Communicates with internal services via gRPC for performance
- **Responsibilities:** Auth validation, response shaping, query complexity limiting, protocol translation

This keeps gRPC's performance benefits for your backend while providing the flexibility of GraphQL or WebSocket to your frontend teams.

---

## gRPC Bidirectional Streaming

Use gRPC for internal service-to-service streaming where type safety, performance, and strict contracts matter.

### Proto Definition

```protobuf
syntax = "proto3";

package collaboration.v1;

service DocumentService {
  // Unary: simple request-response
  rpc GetDocument (GetDocumentRequest) returns (Document);

  // Server streaming: server pushes updates to client
  rpc WatchDocument (WatchRequest) returns (stream DocumentUpdate);

  // Client streaming: client sends a stream of edits
  rpc UploadEdits (stream EditOperation) returns (EditSummary);

  // Bidirectional streaming: both sides stream independently
  rpc CollaborateOnDocument (stream EditOperation) returns (stream DocumentUpdate);
}

message EditOperation {
  string document_id = 1;
  string author_id = 2;
  OperationType type = 3;
  string path = 4;
  bytes value = 5;
  int64 timestamp = 6;
  int32 producer_seq = 7;
}

enum OperationType {
  OPERATION_TYPE_UNSPECIFIED = 0;
  INSERT = 1;
  DELETE = 2;
  UPDATE = 3;
}

message DocumentUpdate {
  string document_id = 1;
  repeated EditOperation operations = 2;
  int64 version = 3;
}
```

### Server Implementation (Go)

```go
func (s *documentServer) CollaborateOnDocument(
    stream pb.DocumentService_CollaborateOnDocumentServer,
) error {
    // Use separate goroutines for reading and writing
    // to prevent one direction from blocking the other
    errCh := make(chan error, 2)

    // Read goroutine: receives edits from client
    go func() {
        for {
            edit, err := stream.Recv()
            if err == io.EOF {
                errCh <- nil
                return
            }
            if err != nil {
                errCh <- err
                return
            }

            // Check deadline — stop processing if expired
            if ctx := stream.Context(); ctx.Err() != nil {
                errCh <- ctx.Err()
                return
            }

            // Process edit and broadcast to other collaborators
            if err := s.processEdit(stream.Context(), edit); err != nil {
                errCh <- err
                return
            }
        }
    }()

    // Write goroutine: pushes updates to client
    go func() {
        updateCh := s.subscribeToDocument(stream.Context())
        for update := range updateCh {
            if err := stream.Send(update); err != nil {
                errCh <- err
                return
            }
        }
        errCh <- nil
    }()

    // Wait for either goroutine to finish
    return <-errCh
}
```

### Deadline Management

Always set explicit deadlines. gRPC propagates deadlines through the `context` — downstream services automatically inherit them:

```go
// Client: set a 30-second deadline for the streaming session
ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
defer cancel()

stream, err := client.CollaborateOnDocument(ctx)
if err != nil {
    log.Fatalf("Failed to start stream: %v", err)
}
```

```go
// Server: check deadline in processing loop
func (s *server) processEdit(ctx context.Context, edit *pb.EditOperation) error {
    // Honor the deadline — stop processing if time is up
    select {
    case <-ctx.Done():
        return status.Error(codes.DeadlineExceeded, "deadline exceeded")
    default:
    }

    // Process the edit...
    return nil
}
```

### Keepalive Configuration

Prevent silent connection drops during idle periods:

```go
import "google.golang.org/grpc/keepalive"

server := grpc.NewServer(
    grpc.KeepaliveParams(keepalive.ServerParameters{
        MaxConnectionIdle:     15 * time.Minute,  // Close idle connections
        MaxConnectionAge:      30 * time.Minute,  // Force reconnection periodically
        MaxConnectionAgeGrace: 5 * time.Second,   // Grace period for in-flight RPCs
        Time:                  10 * time.Second,   // Ping interval
        Timeout:               3 * time.Second,    // Ping timeout
    }),
    grpc.KeepaliveEnforcementPolicy(keepalive.EnforcementPolicy{
        MinTime:             5 * time.Second,  // Min time between pings from client
        PermitWithoutStream: true,             // Allow pings even without active streams
    }),
)
```

### Node.js Client

```javascript
const grpc = require("@grpc/grpc-js");
const protoLoader = require("@grpc/proto-loader");

const packageDef = protoLoader.loadSync("./collaboration.proto");
const proto = grpc.loadPackageDefinition(packageDef).collaboration.v1;

const client = new proto.DocumentService(
  "localhost:50051",
  grpc.credentials.createInsecure()
);

// Bidirectional stream
const stream = client.CollaborateOnDocument();

stream.on("data", (update) => {
  console.log("Received update:", update);
});

stream.on("error", (err) => {
  if (err.code === grpc.status.DEADLINE_EXCEEDED) {
    console.log("Stream deadline exceeded — reconnecting");
    // Implement reconnection with backoff
  }
});

stream.on("end", () => {
  console.log("Stream ended");
});

// Send edits
stream.write({
  documentId: "doc_123",
  authorId: "user_456",
  type: 1, // INSERT
  path: "/title",
  value: Buffer.from("New Title"),
  timestamp: Date.now(),
});
```

---

## GraphQL Subscriptions

Use GraphQL Subscriptions for client-facing real-time features where clients need flexible, typed data shapes.

### Schema Definition

```graphql
type Subscription {
  # Subscribe to document changes — client specifies exactly what fields they need
  documentUpdated(documentId: ID!): DocumentUpdate!
  
  # Subscribe to presence changes in a room
  presenceChanged(roomId: ID!): PresenceEvent!
  
  # Subscribe to new messages in a conversation
  messageReceived(conversationId: ID!): Message!
}

type DocumentUpdate {
  documentId: ID!
  operations: [EditOperation!]!
  version: Int!
  updatedBy: User!
}

type PresenceEvent {
  userId: ID!
  status: PresenceStatus!
  lastSeen: DateTime!
}

enum PresenceStatus {
  ONLINE
  IDLE
  OFFLINE
}
```

### Server Implementation with Redis Pub/Sub

```javascript
import { createServer } from "http";
import { WebSocketServer } from "ws";
import { useServer } from "graphql-ws/lib/use/ws";
import { makeExecutableSchema } from "@graphql-tools/schema";
import { PubSub } from "graphql-redis-subscriptions";
import Redis from "ioredis";

// Distributed pub/sub — works across all server instances
const pubsub = new PubSub({
  publisher: new Redis(process.env.REDIS_URL),
  subscriber: new Redis(process.env.REDIS_URL),
});

const resolvers = {
  Subscription: {
    documentUpdated: {
      subscribe: (_, { documentId }) => {
        // Server-side filtering: client only receives updates for their document
        return pubsub.asyncIterator(`document:${documentId}:updates`);
      },
      resolve: (payload) => payload,
    },

    presenceChanged: {
      subscribe: (_, { roomId }) => {
        return pubsub.asyncIterator(`room:${roomId}:presence`);
      },
    },

    messageReceived: {
      subscribe: (_, { conversationId }, context) => {
        // Verify access before subscribing (BOLA mitigation)
        if (!context.user.conversations.includes(conversationId)) {
          throw new Error("Access denied");
        }
        return pubsub.asyncIterator(`conversation:${conversationId}:messages`);
      },
    },
  },
};

// Publishing events (from any service)
async function onDocumentChanged(documentId, update) {
  await pubsub.publish(`document:${documentId}:updates`, {
    documentUpdated: update,
  });
}

// WebSocket server setup
const schema = makeExecutableSchema({ typeDefs, resolvers });
const server = createServer();
const wsServer = new WebSocketServer({ server, path: "/graphql" });

useServer(
  {
    schema,
    context: async (ctx) => {
      // Authenticate on connection
      const token = ctx.connectionParams?.authToken;
      const user = await validateToken(token);
      return { user };
    },
  },
  wsServer
);
```

### Solving N+1 with DataLoader

When a subscription resolver needs to fetch related data, use DataLoader to batch database queries:

```javascript
import DataLoader from "dataloader";

// Create per-request DataLoader instances
function createLoaders() {
  return {
    userLoader: new DataLoader(async (userIds) => {
      const users = await db.query(
        "SELECT * FROM users WHERE id = ANY($1)",
        [userIds]
      );
      // Return in the same order as the input IDs
      return userIds.map((id) => users.find((u) => u.id === id));
    }),
  };
}

// In subscription resolve
const resolvers = {
  DocumentUpdate: {
    updatedBy: (parent, _, context) => {
      // Batched: if 10 updates arrive, this makes 1 DB query, not 10
      return context.loaders.userLoader.load(parent.authorId);
    },
  },
};
```

### Query Complexity & Depth Limiting

Prevent resource exhaustion from malicious or deeply nested subscriptions:

```javascript
import { createComplexityLimitRule } from "graphql-validation-complexity";
import depthLimit from "graphql-depth-limit";

const validationRules = [
  // Cost-based complexity: assign weights to fields
  createComplexityLimitRule(1000, {
    scalarCost: 1,
    objectCost: 2,
    listFactor: 10,  // Lists multiply the cost of their children
    onCost: (cost) => {
      if (cost > 800) {
        console.warn(`High-complexity query detected: cost=${cost}`);
      }
    },
  }),

  // Depth limiting: prevent infinitely nested queries
  depthLimit(10),
];
```

### Persisted Queries

For production, use persisted queries where the server only executes pre-approved, hash-identified query documents:

```javascript
// Client sends a hash instead of the full query
// POST /graphql
// { "extensions": { "persistedQuery": { "sha256Hash": "abc123..." } } }

const persistedQueries = new Map([
  ["abc123...", "subscription { documentUpdated(documentId: $id) { version operations { type path } } }"],
]);

// Server looks up by hash — rejects unknown queries entirely
```

---

## Protocol Decision Matrix

| Criterion | gRPC Streaming | GraphQL Subscriptions | SSE | WebSocket | WebTransport |
|-----------|---------------|----------------------|-----|-----------|-------------|
| **Boundary** | Service-to-service | Client-to-BFF | Client-to-server | Client-to-server | Client-to-server |
| **Direction** | Bidirectional | Server push (with client query) | Server push | Bidirectional | Bidirectional + datagrams |
| **Data format** | Protobuf (binary) | JSON (flexible shape) | Text (UTF-8) | Text + Binary | Text + Binary |
| **Type safety** | Strict `.proto` contract | Schema-first (SDL) | None (manual) | None (manual) | None (manual) |
| **Browser support** | gRPC-Web proxy required | Native (via WS) | Native | Native | Emerging |
| **Overhead** | Low (HTTP/2 multiplexing) | Medium (resolver execution) | Low | Low | Lowest |
| **Best for** | Internal microservices | Flexible frontend queries | Live feeds, AI tokens | Chat, collab editing | Gaming, HFT, low-latency |

### When to Use What

- **gRPC:** Internal service-to-service where type safety, code generation, and raw throughput matter. Never expose directly to browsers.
- **GraphQL Subscriptions:** Client-facing features where frontend teams need flexible data shapes and self-documenting schemas. Best when aggregating data from multiple backend services.
- **SSE:** Simple server-push (notifications, AI token streaming). Use when you don't need bidirectional communication.
- **WebSocket:** Bidirectional client-server communication (chat, collaborative editing). The workhorse for most real-time features.
- **WebTransport:** Ultra-low-latency applications requiring stream isolation and unreliable datagrams (gaming, HFT, live media).
