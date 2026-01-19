# Hub Server API Reference

## Hub Class

### Constructor

```typescript
new Hub(options: HubOptions)
```

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `port` | number | 9800 | WebSocket server port |
| `executionTimeout` | number | 30000 | Tool execution timeout (ms) |
| `maxSites` | number | 100 | Max concurrent site connections |
| `maxToolsPerSite` | number | 50 | Max tools per site |

### Methods

#### `start(): Promise<void>`
Start the WebSocket server.

#### `stop(): Promise<void>`
Gracefully shut down the server and disconnect all sites.

#### `registerSite(socket: WebSocket, origin: string): string`
Register a new site connection. Returns site ID.

#### `unregisterSite(siteId: string): void`
Remove a site and its tools from the registry.

#### `registerTools(siteId: string, tools: Tool[]): void`
Register tools for a connected site.

#### `getAllTools(): NamespacedTool[]`
Get all tools from all connected sites.

#### `executeToolOnSite(siteId: string, tool: string, args: unknown): Promise<unknown>`
Execute a tool on a specific site.

#### `getSite(siteId: string): ConnectedSite | undefined`
Get site information by ID.

#### `getSiteByOrigin(origin: string): ConnectedSite | undefined`
Get site information by origin.

### Events

| Event | Payload | Description |
|-------|---------|-------------|
| `site:connected` | `{ siteId, origin }` | New site connected |
| `site:disconnected` | `{ siteId, origin }` | Site disconnected |
| `tools:registered` | `{ siteId, tools }` | Site registered tools |
| `tools:updated` | `{ siteId, tools }` | Site updated tools |
| `tool:executed` | `{ siteId, tool, duration }` | Tool execution completed |
| `error` | `{ siteId?, error }` | Error occurred |

---

## ConnectedSite Interface

```typescript
interface ConnectedSite {
  id: string;              // Unique site identifier
  socket: WebSocket;       // WebSocket connection
  origin: string;          // Site origin (e.g., "example.com")
  tools: Tool[];           // Registered tools
  connectedAt: Date;       // Connection timestamp
  metadata?: {             // Optional metadata
    name?: string;
    version?: string;
  };
}
```

---

## Tool Interface

```typescript
interface Tool {
  name: string;            // Tool identifier
  description: string;     // Human-readable description
  inputSchema: {           // JSON Schema for parameters
    type: 'object';
    properties: Record<string, JSONSchema>;
    required?: string[];
  };
}
```

---

## NamespacedTool Interface

```typescript
interface NamespacedTool extends Tool {
  siteId: string;          // Owning site ID
  origin: string;          // Site origin
  originalName: string;    // Original tool name
  namespacedName: string;  // e.g., "example.com/getTodos"
}
```

---

## Error Classes

### HubError
Base error class for all hub errors.

```typescript
class HubError extends Error {
  code: string;
}
```

### SiteNotFoundError
Thrown when referencing a non-existent site.

```typescript
class SiteNotFoundError extends HubError {
  code: 'SITE_NOT_FOUND';
}
```

### ToolNotFoundError
Thrown when referencing a non-existent tool.

```typescript
class ToolNotFoundError extends HubError {
  code: 'TOOL_NOT_FOUND';
}
```

### TimeoutError
Thrown when tool execution times out.

```typescript
class TimeoutError extends HubError {
  code: 'TIMEOUT';
}
```

### ValidationError
Thrown when message validation fails.

```typescript
class ValidationError extends HubError {
  code: 'VALIDATION_ERROR';
}
```

---

## Configuration

### Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `HUB_PORT` | 9800 | WebSocket server port |
| `LOG_LEVEL` | info | Logging level (debug/info/warn/error) |
| `EXECUTION_TIMEOUT` | 30000 | Tool execution timeout (ms) |
| `MAX_SITES` | 100 | Maximum concurrent connections |

### Example Configuration

```typescript
const hub = new Hub({
  port: parseInt(process.env.HUB_PORT || '9800'),
  executionTimeout: parseInt(process.env.EXECUTION_TIMEOUT || '30000'),
  maxSites: parseInt(process.env.MAX_SITES || '100'),
});
```
