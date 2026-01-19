---
name: hub-server
description: WebSocket hub server development for MCP Web Bridge - site connection management, event handling, and MCP server integration. Use when working on packages/server/ code.
allowed-tools: Read, Write, Edit, Bash, Glob, Grep
---

# Hub Server Development

This skill provides knowledge for developing the MCP Web Bridge hub server in `packages/server/`.

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                      Hub Server                              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────────┐ │
│  │ WebSocket   │───▶│    Hub      │───▶│  Tool Registry  │ │
│  │   Server    │    │             │    │                 │ │
│  └─────────────┘    └──────┬──────┘    └────────┬────────┘ │
│                            │                     │          │
│                            ▼                     ▼          │
│                     ┌─────────────┐    ┌─────────────────┐ │
│                     │   Events    │    │   MCP Server    │ │
│                     │             │    │  (stdio/SSE)    │ │
│                     └─────────────┘    └─────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## Key Classes

### Hub Class

The central coordinator managing site connections:

```typescript
import { EventEmitter } from 'events';
import { WebSocket, WebSocketServer } from 'ws';

interface ConnectedSite {
  id: string;
  socket: WebSocket;
  origin: string;
  tools: Tool[];
  connectedAt: Date;
}

class Hub extends EventEmitter {
  private sites: Map<string, ConnectedSite> = new Map();
  private wss: WebSocketServer;

  constructor(options: HubOptions) {
    super();
    this.wss = new WebSocketServer({ port: options.port });
    this.setupConnectionHandler();
  }

  private setupConnectionHandler() {
    this.wss.on('connection', (socket, request) => {
      const origin = request.headers.origin || 'unknown';
      const siteId = this.registerSite(socket, origin);

      socket.on('message', (data) => this.handleMessage(siteId, data));
      socket.on('close', () => this.unregisterSite(siteId));
    });
  }

  registerSite(socket: WebSocket, origin: string): string {
    const id = generateId();
    this.sites.set(id, { id, socket, origin, tools: [], connectedAt: new Date() });
    this.emit('site:connected', { siteId: id, origin });
    return id;
  }

  unregisterSite(siteId: string): void {
    const site = this.sites.get(siteId);
    if (site) {
      this.sites.delete(siteId);
      this.emit('site:disconnected', { siteId, origin: site.origin });
    }
  }
}
```

### Event Types

```typescript
// Emitted events
interface HubEvents {
  'site:connected': { siteId: string; origin: string };
  'site:disconnected': { siteId: string; origin: string };
  'tools:registered': { siteId: string; tools: Tool[] };
  'tools:updated': { siteId: string; tools: Tool[] };
  'tool:executed': { siteId: string; tool: string; duration: number };
  'error': { siteId?: string; error: Error };
}
```

## Message Handling

```typescript
private async handleMessage(siteId: string, data: WebSocket.Data) {
  try {
    const message = JSON.parse(data.toString());

    switch (message.type) {
      case 'register':
        await this.handleRegister(siteId, message);
        break;
      case 'update_tools':
        await this.handleUpdateTools(siteId, message);
        break;
      case 'execute_result':
        await this.handleExecuteResult(siteId, message);
        break;
      default:
        this.logger.warn({ siteId, type: message.type }, 'Unknown message type');
    }
  } catch (error) {
    this.logger.error({ siteId, error }, 'Message handling error');
  }
}
```

## Tool Execution Flow

```typescript
async executeToolOnSite(
  siteId: string,
  toolName: string,
  args: Record<string, unknown>
): Promise<unknown> {
  const site = this.sites.get(siteId);
  if (!site) throw new SiteNotFoundError(siteId);

  const requestId = generateId();

  return new Promise((resolve, reject) => {
    const timeout = setTimeout(() => {
      this.pendingRequests.delete(requestId);
      reject(new TimeoutError(`Tool execution timed out: ${toolName}`));
    }, this.options.executionTimeout);

    this.pendingRequests.set(requestId, { resolve, reject, timeout });

    site.socket.send(JSON.stringify({
      type: 'execute_request',
      requestId,
      tool: toolName,
      args,
    }));
  });
}
```

## Error Handling

Use custom error classes:

```typescript
class HubError extends Error {
  constructor(message: string, public code: string) {
    super(message);
    this.name = 'HubError';
  }
}

class SiteNotFoundError extends HubError {
  constructor(siteId: string) {
    super(`Site not found: ${siteId}`, 'SITE_NOT_FOUND');
  }
}

class TimeoutError extends HubError {
  constructor(message: string) {
    super(message, 'TIMEOUT');
  }
}
```

## Logging Pattern

```typescript
import pino from 'pino';

const logger = pino({
  level: process.env.LOG_LEVEL || 'info',
  formatters: {
    level: (label) => ({ level: label }),
  },
});

// Usage with context
this.logger.info({ siteId, origin, toolCount: tools.length }, 'Site registered tools');
```

For detailed API reference, see [reference.md](reference.md).
