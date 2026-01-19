---
name: client-sdk
description: Browser client SDK development for MCP Web Bridge - WebSocket client, tool definition API, reconnection handling, and browser compatibility. Use when working on packages/client/ code.
allowed-tools: Read, Write, Edit, Bash, Glob, Grep
---

# Client SDK Development

This skill provides knowledge for developing the browser client SDK in `packages/client/`.

## Core API Design

### McpWebBridge Class

```typescript
interface McpWebBridgeOptions {
  hubUrl: string;                    // WebSocket URL (ws:// or wss://)
  reconnect?: boolean;               // Auto-reconnect (default: true)
  reconnectInterval?: number;        // Base retry interval (default: 1000ms)
  reconnectMaxRetries?: number;      // Max retries (default: 10)
  reconnectBackoff?: 'linear' | 'exponential';  // Backoff strategy
}

class McpWebBridge extends EventEmitter {
  constructor(options: McpWebBridgeOptions);

  // Tool definition
  defineTool(tool: ToolDefinition): void;
  removeTool(name: string): void;

  // Connection management
  connect(): Promise<void>;
  disconnect(): void;

  // State
  get isConnected(): boolean;
  get siteId(): string | null;
  get tools(): Tool[];
}
```

### Tool Definition

```typescript
interface ToolDefinition {
  name: string;
  description: string;
  inputSchema: JSONSchema;
  handler: (args: Record<string, unknown>) => Promise<unknown>;
}

// Usage
bridge.defineTool({
  name: 'getCurrentUser',
  description: 'Get the currently logged in user profile',
  inputSchema: {
    type: 'object',
    properties: {},
  },
  handler: async () => {
    return {
      id: user.id,
      name: user.name,
      email: user.email,
    };
  },
});
```

## Connection State Machine

```
┌─────────────┐
│ DISCONNECTED│◀────────────────────────────┐
└──────┬──────┘                              │
       │ connect()                           │
       ▼                                     │
┌─────────────┐                              │
│ CONNECTING  │──── timeout/error ───────────┤
└──────┬──────┘                              │
       │ WebSocket open                      │
       ▼                                     │
┌─────────────┐                              │
│  CONNECTED  │──── close/error ─────────────┤
└──────┬──────┘                              │
       │ ack received                        │
       ▼                                     │
┌─────────────┐                              │
│   READY     │──── close/error ─────────────┘
└─────────────┘
```

## Implementation Patterns

### Event Handling

```typescript
class McpWebBridge extends EventEmitter {
  private emit<K extends keyof BridgeEvents>(
    event: K,
    ...args: Parameters<BridgeEvents[K]>
  ): boolean {
    return super.emit(event, ...args);
  }
}

interface BridgeEvents {
  'connected': (siteId: string) => void;
  'disconnected': (reason?: string) => void;
  'error': (error: Error) => void;
  'tool:executed': (name: string, args: unknown, result: unknown) => void;
  'reconnecting': (attempt: number) => void;
}
```

### Reconnection Logic

```typescript
private scheduleReconnect(): void {
  if (!this.options.reconnect) return;
  if (this.reconnectAttempt >= this.options.reconnectMaxRetries) {
    this.emit('error', new Error('Max reconnection attempts reached'));
    return;
  }

  const delay = this.calculateBackoff();
  this.reconnectAttempt++;
  this.emit('reconnecting', this.reconnectAttempt);

  setTimeout(() => {
    this.connect().catch(() => this.scheduleReconnect());
  }, delay);
}

private calculateBackoff(): number {
  const base = this.options.reconnectInterval;
  if (this.options.reconnectBackoff === 'exponential') {
    return Math.min(base * Math.pow(2, this.reconnectAttempt), 30000);
  }
  return base;
}
```

### Tool Execution Handler

```typescript
private async handleExecuteRequest(message: ExecuteRequestMessage): Promise<void> {
  const tool = this.tools.get(message.tool);

  if (!tool) {
    this.sendResult(message.requestId, false, undefined, {
      code: 'TOOL_NOT_FOUND',
      message: `Tool not found: ${message.tool}`,
    });
    return;
  }

  try {
    const result = await Promise.race([
      tool.handler(message.args),
      this.timeout(this.options.executionTimeout),
    ]);

    this.emit('tool:executed', message.tool, message.args, result);
    this.sendResult(message.requestId, true, result);
  } catch (error) {
    this.sendResult(message.requestId, false, undefined, {
      code: 'EXECUTION_ERROR',
      message: error instanceof Error ? error.message : 'Unknown error',
    });
  }
}
```

## Browser Considerations

### Bundle Configuration (tsup)

```typescript
// tsup.config.ts
export default {
  entry: ['src/index.ts'],
  format: ['esm', 'cjs'],
  dts: true,
  clean: true,
  minify: true,
  target: 'es2020',
  // UMD build for script tag usage
  globalName: 'McpWebBridge',
  platform: 'browser',
};
```

### Script Tag Usage

```html
<script src="https://unpkg.com/@mcp-web-bridge/client"></script>
<script>
  const bridge = new McpWebBridge.McpWebBridge({
    hubUrl: 'ws://localhost:9800',
  });
</script>
```

### ES Module Usage

```typescript
import { McpWebBridge } from '@mcp-web-bridge/client';
```

For usage examples, see [examples.md](examples.md).
