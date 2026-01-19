# MCP Web Bridge Message Formats

This document defines the message formats used between sites and the hub.

## Site → Hub Messages

### Register

Sent when a site first connects to announce its tools.

```typescript
interface RegisterMessage {
  type: 'register';
  tools: Tool[];
  metadata?: {
    name?: string;       // Display name for the site
    version?: string;    // Site/app version
  };
}
```

Example:
```json
{
  "type": "register",
  "tools": [
    {
      "name": "addTodo",
      "description": "Add a new todo item",
      "inputSchema": {
        "type": "object",
        "properties": {
          "title": { "type": "string", "description": "Todo title" }
        },
        "required": ["title"]
      }
    },
    {
      "name": "listTodos",
      "description": "Get all todos",
      "inputSchema": { "type": "object", "properties": {} }
    }
  ],
  "metadata": {
    "name": "Todo App",
    "version": "1.0.0"
  }
}
```

---

### Update Tools

Sent to update the site's tool definitions without reconnecting.

```typescript
interface UpdateToolsMessage {
  type: 'update_tools';
  tools: Tool[];
}
```

Example:
```json
{
  "type": "update_tools",
  "tools": [
    {
      "name": "addTodo",
      "description": "Add a new todo item with priority",
      "inputSchema": {
        "type": "object",
        "properties": {
          "title": { "type": "string" },
          "priority": { "type": "string", "enum": ["low", "medium", "high"] }
        },
        "required": ["title"]
      }
    }
  ]
}
```

---

### Execute Result

Sent in response to an execute request from the hub.

```typescript
interface ExecuteResultMessage {
  type: 'execute_result';
  requestId: string;     // Must match the request
  success: boolean;
  data?: unknown;        // Result if success=true
  error?: {              // Error details if success=false
    code: string;
    message: string;
    details?: unknown;
  };
}
```

Example (Success):
```json
{
  "type": "execute_result",
  "requestId": "req_abc123",
  "success": true,
  "data": {
    "id": 1,
    "title": "Buy groceries",
    "completed": false
  }
}
```

Example (Error):
```json
{
  "type": "execute_result",
  "requestId": "req_abc123",
  "success": false,
  "error": {
    "code": "NOT_FOUND",
    "message": "Todo not found",
    "details": { "id": 999 }
  }
}
```

---

## Hub → Site Messages

### Ack (Acknowledgment)

Sent after a site connects and registers.

```typescript
interface AckMessage {
  type: 'ack';
  siteId: string;        // Assigned site ID
  hubVersion: string;    // Hub protocol version
}
```

Example:
```json
{
  "type": "ack",
  "siteId": "site_7f3a2b1c",
  "hubVersion": "1.0.0"
}
```

---

### Execute Request

Sent when an AI agent calls a tool.

```typescript
interface ExecuteRequestMessage {
  type: 'execute_request';
  requestId: string;     // Unique request ID
  tool: string;          // Tool name (original, not namespaced)
  args: Record<string, unknown>;
}
```

Example:
```json
{
  "type": "execute_request",
  "requestId": "req_abc123",
  "tool": "addTodo",
  "args": {
    "title": "Buy groceries"
  }
}
```

---

### Error

Sent when the hub encounters an error related to the site.

```typescript
interface ErrorMessage {
  type: 'error';
  code: string;
  message: string;
  details?: unknown;
}
```

Example:
```json
{
  "type": "error",
  "code": "INVALID_MESSAGE",
  "message": "Failed to parse message",
  "details": { "raw": "not json" }
}
```

---

## Message Flow Diagrams

### Initial Connection

```
Site                                Hub
  │                                  │
  │──────── WebSocket Connect ──────▶│
  │                                  │
  │◀─────────── Connected ───────────│
  │                                  │
  │──────── register (tools) ───────▶│
  │                                  │
  │◀──────── ack (siteId) ───────────│
  │                                  │
```

### Tool Execution

```
AI Agent           Hub                    Site
    │               │                       │
    │─ tools/call ─▶│                       │
    │               │── execute_request ───▶│
    │               │                       │
    │               │◀── execute_result ────│
    │◀── result ────│                       │
    │               │                       │
```

### Tool Update

```
Site                                Hub
  │                                  │
  │──── update_tools (tools) ───────▶│
  │                                  │
  │               (Hub updates registry)
  │                                  │
  │               (No response required)
```

---

## Validation Schemas (Zod)

```typescript
import { z } from 'zod';

const ToolSchema = z.object({
  name: z.string().min(1).max(64).regex(/^[a-zA-Z][a-zA-Z0-9_]*$/),
  description: z.string().min(1).max(1024),
  inputSchema: z.object({
    type: z.literal('object'),
    properties: z.record(z.unknown()),
    required: z.array(z.string()).optional(),
  }),
});

const RegisterMessageSchema = z.object({
  type: z.literal('register'),
  tools: z.array(ToolSchema),
  metadata: z.object({
    name: z.string().optional(),
    version: z.string().optional(),
  }).optional(),
});

const ExecuteResultSchema = z.object({
  type: z.literal('execute_result'),
  requestId: z.string(),
  success: z.boolean(),
  data: z.unknown().optional(),
  error: z.object({
    code: z.string(),
    message: z.string(),
    details: z.unknown().optional(),
  }).optional(),
});
```
