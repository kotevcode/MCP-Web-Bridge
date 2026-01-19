---
name: mcp-protocol
description: Model Context Protocol (MCP) knowledge - tool definitions, message formats, transport mechanisms, and SDK usage. Use when working with MCP protocol implementation or tool schemas.
allowed-tools: Read, Grep, Glob, WebFetch, WebSearch
---

# MCP Protocol Knowledge

This skill provides knowledge about the Model Context Protocol (MCP) for implementing MCP servers and clients.

## Protocol Overview

MCP enables AI agents to interact with external tools and data sources. It uses JSON-RPC 2.0 for message format and supports multiple transports.

```
┌──────────────┐                    ┌──────────────┐
│   AI Agent   │◀──── MCP ─────────▶│  MCP Server  │
│   (Claude)   │   (JSON-RPC 2.0)   │              │
└──────────────┘                    └──────────────┘
```

## Tool Definition

Tools are functions that AI agents can discover and call:

```typescript
interface Tool {
  name: string;              // Unique identifier (lowercase, no spaces)
  description: string;       // Clear description for AI to understand usage
  inputSchema: {             // JSON Schema for parameters
    type: 'object';
    properties: {
      [key: string]: {
        type: string;
        description?: string;
        enum?: string[];
      };
    };
    required?: string[];
  };
}
```

### Good Tool Descriptions

```typescript
// GOOD: Clear, actionable description
{
  name: 'searchProducts',
  description: 'Search for products by name, category, or price range. Returns matching products with ID, name, price, and availability.',
  inputSchema: {
    type: 'object',
    properties: {
      query: { type: 'string', description: 'Search query (product name or keywords)' },
      category: { type: 'string', enum: ['electronics', 'clothing', 'home'] },
      maxPrice: { type: 'number', description: 'Maximum price filter' }
    },
    required: ['query']
  }
}

// BAD: Vague, unhelpful description
{
  name: 'search',
  description: 'Searches stuff',
  inputSchema: { type: 'object', properties: { q: { type: 'string' } } }
}
```

## Transport Mechanisms

### 1. stdio (Standard I/O)

Used by Claude Desktop. Messages are newline-delimited JSON on stdin/stdout.

```typescript
import { Server } from '@modelcontextprotocol/sdk/server/index.js';
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js';

const server = new Server({ name: 'my-server', version: '1.0.0' }, { capabilities: { tools: {} } });
const transport = new StdioServerTransport();
await server.connect(transport);
```

### 2. SSE (Server-Sent Events)

HTTP-based transport for web clients:

```typescript
import { SSEServerTransport } from '@modelcontextprotocol/sdk/server/sse.js';

const transport = new SSEServerTransport('/sse', response);
await server.connect(transport);
```

### 3. WebSocket

Full-duplex real-time communication (custom implementation needed).

## SDK Usage

### Creating an MCP Server

```typescript
import { Server } from '@modelcontextprotocol/sdk/server/index.js';
import { ListToolsRequestSchema, CallToolRequestSchema } from '@modelcontextprotocol/sdk/types.js';

const server = new Server(
  { name: 'mcp-web-bridge', version: '1.0.0' },
  { capabilities: { tools: {} } }
);

// List available tools
server.setRequestHandler(ListToolsRequestSchema, async () => {
  return {
    tools: [
      {
        name: 'example.com/getTodos',
        description: 'Get all todos from example.com',
        inputSchema: { type: 'object', properties: {} }
      }
    ]
  };
});

// Handle tool execution
server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const { name, arguments: args } = request.params;

  // Route to appropriate site and execute
  const result = await executeToolOnSite(name, args);

  return {
    content: [{ type: 'text', text: JSON.stringify(result) }]
  };
});
```

## JSON-RPC 2.0 Format

### Request
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "example.com/getTodos",
    "arguments": {}
  }
}
```

### Response (Success)
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "content": [{ "type": "text", "text": "[{\"id\":1,\"title\":\"Buy milk\"}]" }]
  }
}
```

### Response (Error)
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "error": {
    "code": -32602,
    "message": "Invalid params",
    "data": { "details": "Missing required parameter: userId" }
  }
}
```

## Error Codes

| Code | Meaning |
|------|---------|
| -32700 | Parse error |
| -32600 | Invalid request |
| -32601 | Method not found |
| -32602 | Invalid params |
| -32603 | Internal error |

For message format details, see [message-formats.md](message-formats.md).
