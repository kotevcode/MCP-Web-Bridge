# Client SDK Examples

## Basic Usage

### Minimal Setup

```typescript
import { McpWebBridge } from '@mcp-web-bridge/client';

const bridge = new McpWebBridge({
  hubUrl: 'ws://localhost:9800',
});

bridge.defineTool({
  name: 'ping',
  description: 'Returns pong',
  inputSchema: { type: 'object', properties: {} },
  handler: async () => 'pong',
});

await bridge.connect();
console.log('Connected with site ID:', bridge.siteId);
```

---

## Common Patterns

### CRUD Operations

```typescript
// Define a set of CRUD tools for a resource
const items: Map<string, Item> = new Map();

bridge.defineTool({
  name: 'createItem',
  description: 'Create a new item. Returns the created item with its ID.',
  inputSchema: {
    type: 'object',
    properties: {
      title: { type: 'string', description: 'Item title' },
      data: { type: 'object', description: 'Additional item data' },
    },
    required: ['title'],
  },
  handler: async ({ title, data }) => {
    const id = crypto.randomUUID();
    const item = { id, title, data, createdAt: new Date().toISOString() };
    items.set(id, item);
    return item;
  },
});

bridge.defineTool({
  name: 'getItem',
  description: 'Get an item by ID. Returns null if not found.',
  inputSchema: {
    type: 'object',
    properties: {
      id: { type: 'string', description: 'Item ID' },
    },
    required: ['id'],
  },
  handler: async ({ id }) => items.get(id) || null,
});

bridge.defineTool({
  name: 'listItems',
  description: 'Get all items. Returns an array of items.',
  inputSchema: { type: 'object', properties: {} },
  handler: async () => Array.from(items.values()),
});

bridge.defineTool({
  name: 'deleteItem',
  description: 'Delete an item by ID. Returns true if deleted, false if not found.',
  inputSchema: {
    type: 'object',
    properties: {
      id: { type: 'string', description: 'Item ID' },
    },
    required: ['id'],
  },
  handler: async ({ id }) => items.delete(id),
});
```

---

### With Authentication

```typescript
const bridge = new McpWebBridge({
  hubUrl: 'wss://hub.example.com',
});

// Tool that requires auth
bridge.defineTool({
  name: 'getUserProfile',
  description: 'Get the current user profile. Requires authentication.',
  inputSchema: { type: 'object', properties: {} },
  handler: async () => {
    if (!auth.isLoggedIn()) {
      throw new Error('User not authenticated');
    }
    return {
      id: auth.user.id,
      name: auth.user.name,
      email: auth.user.email,
    };
  },
});
```

---

### Event Handling

```typescript
const bridge = new McpWebBridge({
  hubUrl: 'ws://localhost:9800',
  reconnect: true,
});

// Connection events
bridge.on('connected', (siteId) => {
  console.log('Connected to hub with ID:', siteId);
  updateUI('connected');
});

bridge.on('disconnected', (reason) => {
  console.log('Disconnected:', reason);
  updateUI('disconnected');
});

bridge.on('reconnecting', (attempt) => {
  console.log(`Reconnecting... attempt ${attempt}`);
  updateUI('reconnecting');
});

bridge.on('error', (error) => {
  console.error('Bridge error:', error);
  showError(error.message);
});

// Tool execution events
bridge.on('tool:executed', (name, args, result) => {
  console.log(`Tool ${name} executed:`, { args, result });
  logToolCall(name, args, result);
});

await bridge.connect();
```

---

### Dynamic Tool Registration

```typescript
// Add tool after connection
bridge.defineTool({
  name: 'newFeature',
  description: 'A newly added feature',
  inputSchema: { type: 'object', properties: {} },
  handler: async () => 'New feature result',
});

// Remove tool
bridge.removeTool('oldFeature');

// Note: Tool updates are automatically sent to the hub
```

---

### Error Handling

```typescript
bridge.defineTool({
  name: 'riskyOperation',
  description: 'An operation that might fail',
  inputSchema: {
    type: 'object',
    properties: {
      shouldFail: { type: 'boolean' },
    },
  },
  handler: async ({ shouldFail }) => {
    if (shouldFail) {
      // Throwing an error sends it back to the AI agent
      throw new Error('Operation failed as requested');
    }
    return { success: true };
  },
});
```

---

### With React

```tsx
import { McpWebBridge } from '@mcp-web-bridge/client';
import { useEffect, useState } from 'react';

function useMcpBridge(hubUrl: string) {
  const [bridge] = useState(() => new McpWebBridge({ hubUrl }));
  const [isConnected, setIsConnected] = useState(false);
  const [siteId, setSiteId] = useState<string | null>(null);

  useEffect(() => {
    bridge.on('connected', (id) => {
      setIsConnected(true);
      setSiteId(id);
    });

    bridge.on('disconnected', () => {
      setIsConnected(false);
      setSiteId(null);
    });

    bridge.connect();

    return () => {
      bridge.disconnect();
    };
  }, [bridge]);

  return { bridge, isConnected, siteId };
}

function App() {
  const { bridge, isConnected, siteId } = useMcpBridge('ws://localhost:9800');

  useEffect(() => {
    if (!bridge) return;

    bridge.defineTool({
      name: 'getAppState',
      description: 'Get the current React app state',
      inputSchema: { type: 'object', properties: {} },
      handler: async () => ({ /* app state */ }),
    });
  }, [bridge]);

  return (
    <div>
      <p>Status: {isConnected ? `Connected (${siteId})` : 'Disconnected'}</p>
    </div>
  );
}
```

---

### Script Tag (No Build)

```html
<!DOCTYPE html>
<html>
<head>
  <script src="https://unpkg.com/@mcp-web-bridge/client"></script>
</head>
<body>
  <div id="status">Connecting...</div>

  <script>
    const bridge = new McpWebBridge.McpWebBridge({
      hubUrl: 'ws://localhost:9800',
    });

    bridge.defineTool({
      name: 'getPageTitle',
      description: 'Get the current page title',
      inputSchema: { type: 'object', properties: {} },
      handler: async () => document.title,
    });

    bridge.on('connected', (siteId) => {
      document.getElementById('status').textContent = 'Connected: ' + siteId;
    });

    bridge.connect();
  </script>
</body>
</html>
```
