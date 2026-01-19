---
name: websocket-debugging
description: Debugging WebSocket connections in MCP Web Bridge - connection issues, message inspection, common error patterns, and testing tools. Use when troubleshooting WebSocket problems.
allowed-tools: Read, Bash, Grep, Glob
---

# WebSocket Debugging

This skill provides techniques for debugging WebSocket connections in MCP Web Bridge.

## Common Issues

### 1. Connection Refused

**Symptom:** Client can't connect to hub

**Checklist:**
- [ ] Is the hub server running?
- [ ] Correct port? (default: 9800)
- [ ] Firewall blocking the port?
- [ ] Using correct protocol? (ws:// for local, wss:// for production)

**Debug steps:**
```bash
# Check if hub is listening
lsof -i :9800
# or
netstat -tlnp | grep 9800

# Test connection with wscat
npx wscat -c ws://localhost:9800

# Check hub logs
tail -f hub.log
```

### 2. Connection Drops Immediately

**Symptom:** WebSocket connects then immediately closes

**Common causes:**
- Origin validation failing
- Invalid handshake
- Server error during connection setup

**Debug:**
```bash
# Connect with verbose output
npx wscat -c ws://localhost:9800 -v

# Check close code and reason
# 1008 = Policy violation (origin rejected)
# 1011 = Server error
# 1006 = Abnormal closure (no close frame)
```

### 3. Messages Not Received

**Symptom:** Client sends message but hub doesn't respond

**Checklist:**
- [ ] Message is valid JSON?
- [ ] Message has correct `type` field?
- [ ] Message matches expected schema?

**Debug:**
```typescript
// Add message logging on client
socket.addEventListener('message', (event) => {
  console.log('Received:', event.data);
});

// Add send logging
const originalSend = socket.send.bind(socket);
socket.send = (data) => {
  console.log('Sending:', data);
  return originalSend(data);
};
```

### 4. Tool Execution Timeout

**Symptom:** Tool calls timeout without response

**Common causes:**
- Handler throws unhandled error
- Handler takes too long
- Message routing issue

**Debug:**
```typescript
// Add timing to handler
bridge.defineTool({
  name: 'slowTool',
  handler: async (args) => {
    console.time('slowTool');
    try {
      const result = await doWork(args);
      return result;
    } finally {
      console.timeEnd('slowTool');
    }
  }
});
```

## Debugging Tools

### wscat

Interactive WebSocket client:

```bash
# Install
npm install -g wscat

# Connect
npx wscat -c ws://localhost:9800

# Send message (once connected)
> {"type":"register","tools":[]}
```

### websocat

More powerful alternative:

```bash
# Install (varies by OS)
brew install websocat  # macOS
cargo install websocat # with Rust

# Connect with auto-reconnect
websocat -t ws://localhost:9800 --ping-interval 30

# Connect and send from file
cat message.json | websocat ws://localhost:9800
```

### Browser DevTools

1. Open DevTools (F12)
2. Go to Network tab
3. Filter by "WS"
4. Click on WebSocket connection
5. View Messages tab for all sent/received messages

### Custom Debug Proxy

Create a proxy to inspect all messages:

```typescript
// debug-proxy.ts
import { WebSocketServer, WebSocket } from 'ws';

const proxyPort = 9801;
const hubUrl = 'ws://localhost:9800';

const wss = new WebSocketServer({ port: proxyPort });

wss.on('connection', (clientWs) => {
  const hubWs = new WebSocket(hubUrl);

  hubWs.on('open', () => {
    console.log('[PROXY] Connected to hub');
  });

  clientWs.on('message', (data) => {
    console.log('[CLIENT → HUB]', data.toString());
    hubWs.send(data);
  });

  hubWs.on('message', (data) => {
    console.log('[HUB → CLIENT]', data.toString());
    clientWs.send(data);
  });

  clientWs.on('close', () => hubWs.close());
  hubWs.on('close', () => clientWs.close());
});

console.log(`Debug proxy listening on ws://localhost:${proxyPort}`);
```

## Message Validation

### Quick JSON Check

```bash
# Validate JSON syntax
echo '{"type":"register","tools":[]}' | jq .

# Pretty print
echo '{"type":"register","tools":[{"name":"test","description":"Test","inputSchema":{"type":"object"}}]}' | jq .
```

### Schema Validation

```typescript
import { z } from 'zod';

const MessageSchema = z.discriminatedUnion('type', [
  z.object({
    type: z.literal('register'),
    tools: z.array(ToolSchema),
  }),
  z.object({
    type: z.literal('execute_result'),
    requestId: z.string(),
    success: z.boolean(),
    data: z.unknown().optional(),
    error: z.object({
      code: z.string(),
      message: z.string(),
    }).optional(),
  }),
]);

// Usage
try {
  const parsed = MessageSchema.parse(JSON.parse(rawMessage));
  console.log('Valid message:', parsed);
} catch (error) {
  console.error('Invalid message:', error.errors);
}
```

## Log Levels

Configure hub logging for debugging:

```bash
# Set log level
LOG_LEVEL=debug npm run start

# Log levels:
# - trace: Everything including message payloads
# - debug: Connection events, message types
# - info: Connections, tool registrations (default)
# - warn: Recoverable errors
# - error: Unrecoverable errors
```

## Health Check Endpoint

Add HTTP health check alongside WebSocket:

```typescript
import http from 'http';

const server = http.createServer((req, res) => {
  if (req.url === '/health') {
    res.writeHead(200, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify({
      status: 'ok',
      connectedSites: hub.getSiteCount(),
      toolCount: registry.getAll().length,
      uptime: process.uptime(),
    }));
  }
});

// Usage
curl http://localhost:9800/health
```

## Troubleshooting Checklist

When debugging, work through this checklist:

1. **Hub running?** - `lsof -i :9800`
2. **Can connect?** - `npx wscat -c ws://localhost:9800`
3. **Handshake works?** - Send `{"type":"register","tools":[]}`
4. **Get ack?** - Should receive `{"type":"ack","siteId":"..."}`
5. **Tools registered?** - Check hub logs or admin endpoint
6. **MCP sees tools?** - Test with Claude Desktop
7. **Execution works?** - Trigger tool and check both ends
