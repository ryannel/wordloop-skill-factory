# AI & MCP Integration

Deep reference for designing real-time systems that serve autonomous AI agents via the Model Context Protocol (MCP). MCP defines how AI agents (such as Claude, Gemini, or custom orchestrators) interface with internal systems safely through persistent connections.

## Table of Contents

- [MCP Architecture](#mcp-architecture)
- [Resource Design](#resource-design)
- [Tool Design](#tool-design)
- [Prompt Templates](#prompt-templates)
- [Transport Selection](#transport-selection)
- [Security for Autonomous Agents](#security-for-autonomous-agents)
- [Schema Validation & Injection Prevention](#schema-validation--injection-prevention)
- [API Discoverability for Agents](#api-discoverability-for-agents)

---

## MCP Architecture

### The Three-Layer Model

```
┌─────────────┐
│  AI Agent   │  (Claude, Gemini, custom orchestrator)
│  (Reasoning)│
└──────┬──────┘
       │ MCP Protocol
┌──────▼──────┐
│  MCP Client │  (Host application — manages agent context)
│  (Host)     │
└──────┬──────┘
       │ SSE / WebTransport / stdio
┌──────▼──────┐
│  MCP Server │  (Your application — exposes capabilities)
│  (Provider) │
└─────────────┘
```

The AI agent talks to the MCP Client (the Host). The Host opens a persistent connection to the MCP Server. The server exposes three primitive types:

| Primitive | Description | Analogy |
|-----------|-------------|---------|
| **Resources** | Read-only data access | GET endpoints |
| **Tools** | Executable functions with side effects | POST/PUT endpoints |
| **Prompts** | Reusable instruction templates | Parameterized prompt library |

### Key Principle

MCP servers are designed to be **tool-friendly** — every capability is self-describing with strict schemas so that an AI agent can discover, understand, and invoke them without human intervention.

---

## Resource Design

Resources expose read-only data that the agent can query to build context. Design resources to be discoverable and self-describing:

### Resource Definition

```json
{
  "resources": [
    {
      "uri": "database://orders/recent",
      "name": "Recent Orders",
      "description": "Last 100 orders with status, customer, and total amount",
      "mimeType": "application/json",
      "schema": {
        "type": "array",
        "items": {
          "$ref": "#/components/schemas/Order"
        }
      }
    },
    {
      "uri": "file://docs/api-reference",
      "name": "API Reference Documentation",
      "description": "Complete API documentation in markdown format",
      "mimeType": "text/markdown"
    }
  ]
}
```

### Design Principles for Resources

- **Descriptive names and descriptions** — The agent relies on these to decide which resource to read
- **Scoped access** — Each resource returns a bounded, relevant dataset (never unbounded queries)
- **MIME types** — Always specify `mimeType` so the agent knows how to parse the response
- **Schema definitions** — Include JSON Schema for structured resources so the agent can reason about fields

---

## Tool Design

Tools are executable functions the agent can invoke. They are the most security-sensitive primitive because they have side effects.

### Tool Definition

```json
{
  "tools": [
    {
      "name": "query_database",
      "description": "Execute a read-only SQL query against the analytics database. Returns up to 1000 rows. Use for data analysis and reporting tasks.",
      "inputSchema": {
        "type": "object",
        "required": ["query"],
        "properties": {
          "query": {
            "type": "string",
            "description": "SQL SELECT query. Only SELECT statements are allowed.",
            "pattern": "^SELECT\\s",
            "maxLength": 2000
          },
          "limit": {
            "type": "integer",
            "default": 100,
            "maximum": 1000,
            "description": "Maximum rows to return"
          }
        }
      }
    },
    {
      "name": "create_ticket",
      "description": "Create a support ticket in the ticketing system. Returns the ticket ID and URL.",
      "inputSchema": {
        "type": "object",
        "required": ["title", "description", "priority"],
        "properties": {
          "title": {
            "type": "string",
            "maxLength": 200
          },
          "description": {
            "type": "string",
            "maxLength": 5000
          },
          "priority": {
            "type": "string",
            "enum": ["low", "medium", "high", "critical"]
          },
          "assignee": {
            "type": "string",
            "description": "User ID of the assignee (optional)"
          }
        }
      }
    }
  ]
}
```

### Design Principles for Tools

- **Clear, unambiguous descriptions** — The agent uses the description to decide when and how to invoke the tool
- **Strict input schemas** — Use `required`, `enum`, `pattern`, `maxLength`, `maximum` constraints aggressively
- **Least privilege** — Each tool should have the minimum permissions needed. A "query database" tool should only allow SELECT statements
- **Idempotency markers** — Tools that create or modify data should accept and respect `idempotencyKey` parameters
- **Error responses** — Return structured errors with actionable messages so the agent can retry or adjust parameters

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "The query contains a DELETE statement. Only SELECT queries are permitted.",
    "retryable": false
  }
}
```

---

## Prompt Templates

Prompts are reusable instruction templates that the agent or user can invoke with parameters:

```json
{
  "prompts": [
    {
      "name": "analyze_performance",
      "description": "Generate a performance analysis report for a service over a time range",
      "arguments": [
        {
          "name": "service_name",
          "description": "Name of the service to analyze",
          "required": true
        },
        {
          "name": "time_range",
          "description": "Time range for analysis (e.g., '24h', '7d', '30d')",
          "required": true
        }
      ]
    }
  ]
}
```

---

## Transport Selection

MCP supports multiple transport mechanisms. Choose based on deployment context:

| Transport | Use Case | Characteristics |
|-----------|----------|-----------------|
| **stdio** | Local development, CLI tools | Synchronous, single-user, lowest latency |
| **SSE** | Web-hosted servers, multi-user | Server-push events, HTTP-native, auto-reconnect |
| **WebTransport** | High-performance, low-latency | Multi-stream, 0-RTT resume, binary support |

### SSE Transport (Recommended for Web)

```javascript
import express from "express";

const app = express();

// MCP SSE endpoint
app.get("/mcp/events", (req, res) => {
  res.setHeader("Content-Type", "text/event-stream");
  res.setHeader("Cache-Control", "no-cache");
  res.setHeader("Connection", "keep-alive");

  // Send capability advertisement
  const capabilities = {
    resources: true,
    tools: true,
    prompts: true,
  };

  res.write(`event: capabilities\n`);
  res.write(`data: ${JSON.stringify(capabilities)}\n\n`);

  // Handle tool invocations via POST and respond via SSE
  const channel = `mcp:${req.query.sessionId}`;

  subscriber.subscribe(channel, (message) => {
    const response = JSON.parse(message);
    res.write(`event: tool_response\n`);
    res.write(`id: ${response.id}\n`);
    res.write(`data: ${JSON.stringify(response)}\n\n`);
  });

  req.on("close", () => {
    subscriber.unsubscribe(channel);
    res.end();
  });
});

// MCP tool invocation endpoint
app.post("/mcp/tools/:toolName", express.json(), async (req, res) => {
  const { toolName } = req.params;
  const { arguments: args, sessionId, requestId } = req.body;

  // Validate and execute
  const result = await executeTool(toolName, args);

  // Publish result to the SSE channel
  await publisher.publish(`mcp:${sessionId}`, JSON.stringify({
    id: requestId,
    result,
  }));

  res.status(202).json({ requestId });
});
```

---

## Security for Autonomous Agents

AI agents operate autonomously and can be manipulated through prompt injection. Apply defense-in-depth:

### Principle of Least Privilege

```javascript
// Per-agent capability scoping
const agentPermissions = {
  "agent-analytics": {
    resources: ["database://orders/recent", "database://metrics/*"],
    tools: ["query_database"],     // Read-only tools only
    prompts: ["analyze_performance"],
  },
  "agent-support": {
    resources: ["database://tickets/*"],
    tools: ["query_database", "create_ticket", "update_ticket"],
    prompts: ["analyze_performance", "draft_response"],
  },
};

function authorizeToolCall(agentId, toolName) {
  const perms = agentPermissions[agentId];
  if (!perms || !perms.tools.includes(toolName)) {
    throw new Error(`Agent ${agentId} not authorized for tool ${toolName}`);
  }
}
```

### Rate Limiting for Agents

Agents can issue rapid-fire tool calls. Apply aggressive rate limits:

```javascript
const AGENT_RATE_LIMITS = {
  "query_database": { maxPerMinute: 10, maxPerHour: 100 },
  "create_ticket": { maxPerMinute: 5, maxPerHour: 50 },
};
```

### Audit Trail

Every agent action must be logged with full context for compliance and debugging:

```javascript
function auditAgentAction(agentId, toolName, args, result) {
  auditLog.info("agent.tool_invocation", {
    agentId,
    tool: toolName,
    arguments: sanitizeForLogging(args),
    resultStatus: result.error ? "error" : "success",
    timestamp: Date.now(),
    traceId: getCurrentTraceId(),
  });
}
```

---

## Schema Validation & Injection Prevention

The Tool execution layer is the most critical security boundary. Validate aggressively to prevent prompt injection attacks from autonomous agents.

### Input Sanitization

```javascript
import Ajv from "ajv";
import addFormats from "ajv-formats";

const ajv = new Ajv({ allErrors: true, coerceTypes: false });
addFormats(ajv);

// Pre-compile tool schemas for performance
const toolValidators = {};
for (const tool of toolDefinitions) {
  toolValidators[tool.name] = ajv.compile(tool.inputSchema);
}

async function executeTool(toolName, args) {
  const validate = toolValidators[toolName];

  if (!validate) {
    throw new Error(`Unknown tool: ${toolName}`);
  }

  if (!validate(args)) {
    return {
      error: {
        code: "VALIDATION_ERROR",
        message: "Invalid tool arguments",
        details: validate.errors,
        retryable: true,
      },
    };
  }

  // Additional semantic validation beyond schema
  if (toolName === "query_database") {
    // Prevent SQL injection even with schema validation
    if (/;\s*(DROP|DELETE|UPDATE|INSERT|ALTER)/i.test(args.query)) {
      return {
        error: {
          code: "FORBIDDEN_QUERY",
          message: "Only SELECT queries are allowed",
          retryable: false,
        },
      };
    }
  }

  return await toolHandlers[toolName](args);
}
```

### Output Sanitization

Sanitize tool outputs before returning them to the agent to prevent data leakage:

```javascript
function sanitizeToolOutput(toolName, result) {
  // Remove internal fields the agent should not see
  const { _internalId, _auditLog, ...safeResult } = result;

  // Truncate large outputs to prevent context overflow
  const serialized = JSON.stringify(safeResult);
  if (serialized.length > 50000) {
    return {
      ...safeResult,
      _truncated: true,
      _message: "Output truncated to 50KB. Use pagination for large results.",
    };
  }

  return safeResult;
}
```

---

## API Discoverability for Agents

For APIs consumed by LLMs and autonomous agents, serve machine-readable specs at well-known paths:

- `/.well-known/asyncapi.yaml` — AsyncAPI spec for event-driven endpoints
- `/openapi.json` — OpenAPI spec for REST endpoints
- `/llms.txt` — Concise, machine-readable API summary optimized for LLM context windows

The `llms.txt` file should include available endpoints, authentication requirements, and rate limits in a token-efficient format. This enables AI agents to discover and integrate with your API without consuming excessive context.
