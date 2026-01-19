# MCP Web Bridge - Implementation Plan
 
> A shared MCP server that allows websites to connect via WebSocket and expose their tools to AI agents.
 
## Vision
 
Create an open-source protocol and reference implementation that enables any website to expose MCP-compatible tools through a shared hub. AI agents connect once to the hub and gain access to tools from all connected websites.
 
```
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│  Website A  │  │  Website B  │  │  Website C  │
│  (Browser)  │  │  (Browser)  │  │  (Browser)  │
└──────┬──────┘  └──────┬──────┘  └──────┬──────┘
       │ WS             │ WS             │ WS
       └────────────────┼────────────────┘
                        ▼
              ┌───────────────────┐
              │   MCP Web Bridge  │
              │   (Hub Server)    │
              └─────────┬─────────┘
                        │ MCP (stdio/SSE/WebSocket)
                        ▼
              ┌───────────────────┐
              │   AI Agent        │
              │   (Claude, etc)   │
              └───────────────────┘
```
 
---
 
## Phase 1: Core Infrastructure (MVP)
 
### Goals
- Working hub server that accepts website connections
- Basic MCP server exposing aggregated tools
- Simple client SDK for websites
- Proof of concept with example sites
 
### 1.1 Hub Server (`packages/server`)
 
```
packages/server/
├── src/
│   ├── index.ts              # Entry point
│   ├── hub.ts                # WebSocket hub for website connections
│   ├── mcp-server.ts         # MCP server (stdio + SSE transports)
│   ├── registry.ts           # Tool registry (aggregates tools from sites)
│   ├── types.ts              # Shared types
│   └── utils/
│       ├── logger.ts         # Structured logging
│       └── id.ts             # ID generation
├── tests/
│   ├── hub.test.ts
│   ├── registry.test.ts
│   └── integration.test.ts
├── package.json
└── tsconfig.json
```
 
**Key Components:**
 
```typescript
// hub.ts - Core hub logic
interface ConnectedSite {
  id: string;
  socket: WebSocket;
  origin: string;
  tools: Tool[];
  connectedAt: Date;
}
 
class Hub extends EventEmitter {
  private sites: Map<string, ConnectedSite>;
 
  // Site management
  registerSite(socket: WebSocket, origin: string): string;
  unregisterSite(siteId: string): void;
 
  // Tool management
  registerTools(siteId: string, tools: Tool[]): void;
  getAllTools(): Tool[];  // Namespaced: "origin.com/toolName"
 
  // Execution
  executeToolOnSite(siteId: string, toolName: string, args: unknown): Promise<unknown>;
}
```
 
```typescript
// registry.ts - Tool aggregation
interface NamespacedTool extends Tool {
  siteId: string;
  origin: string;
  originalName: string;
  namespacedName: string;  // "example.com/sendEmail"
}
 
class ToolRegistry {
  private tools: Map<string, NamespacedTool>;
 
  register(siteId: string, origin: string, tools: Tool[]): void;
  unregister(siteId: string): void;
  getAll(): Tool[];
  resolve(namespacedName: string): { siteId: string; toolName: string } | undefined;
}
```
 
### 1.2 Client SDK (`packages/client`)
 
```
packages/client/
├── src/
│   ├── index.ts              # Main export
│   ├── client.ts             # WebSocket client
│   ├── tool-builder.ts       # Fluent API for defining tools
│   └── types.ts
├── tests/
│   ├── client.test.ts
│   └── tool-builder.test.ts
├── package.json
└── tsconfig.json
```
 
**API Design:**
 
```typescript
// Simple usage
import { McpWebBridge } from '@mcp-web-bridge/client';
 
const bridge = new McpWebBridge({
  hubUrl: 'ws://localhost:9800',
});
 
// Define tools
bridge.defineTool({
  name: 'getCurrentUser',
  description: 'Get the currently logged in user',
  inputSchema: { type: 'object', properties: {} },
  handler: async () => {
    return { user: currentUser };
  },
});
 
bridge.defineTool({
  name: 'sendMessage',
  description: 'Send a message to a user',
  inputSchema: {
    type: 'object',
    properties: {
      userId: { type: 'string' },
      message: { type: 'string' },
    },
    required: ['userId', 'message'],
  },
  handler: async ({ userId, message }) => {
    await api.sendMessage(userId, message);
    return { success: true };
  },
});
 
// Connect
await bridge.connect();
```
 
### 1.3 Protocol Specification (`docs/protocol.md`)
 
```typescript
// Site → Hub messages
interface RegisterMessage {
  type: 'register';
  tools: Tool[];
}
 
interface UpdateToolsMessage {
  type: 'update_tools';
  tools: Tool[];
}
 
interface ExecuteResultMessage {
  type: 'execute_result';
  requestId: string;
  success: boolean;
  data?: unknown;
  error?: string;
}
 
// Hub → Site messages
interface ExecuteRequestMessage {
  type: 'execute_request';
  requestId: string;
  tool: string;
  args: Record<string, unknown>;
}
 
interface AckMessage {
  type: 'ack';
  siteId: string;
}
```
 
---
 
## Phase 2: Examples & POC
 
### 2.1 Example: Todo App (`examples/todo-app`)
 
A simple todo app that exposes tools:
- `addTodo(title)` - Add a new todo
- `listTodos()` - List all todos
- `completeTodo(id)` - Mark todo as complete
- `deleteTodo(id)` - Delete a todo
 
```
examples/todo-app/
├── index.html
├── app.js              # Todo app logic
├── bridge.js           # Integration with MCP Web Bridge
└── README.md
```
 
### 2.2 Example: E-commerce (`examples/ecommerce`)
 
Mock e-commerce site exposing:
- `searchProducts(query)` - Search products
- `getProductDetails(id)` - Get product info
- `addToCart(productId, quantity)` - Add to cart
- `getCart()` - Get current cart
- `checkout()` - Process checkout
 
### 2.3 Example: Calendar (`examples/calendar`)
 
Calendar app exposing:
- `getEvents(startDate, endDate)` - List events
- `createEvent(title, start, end)` - Create event
- `updateEvent(id, updates)` - Update event
- `deleteEvent(id)` - Delete event
 
### 2.4 POC: Claude Desktop Integration
 
Step-by-step guide to:
1. Run the hub server
2. Open example sites in browser
3. Configure Claude Desktop to use the MCP server
4. Interact with website tools through Claude
 
---
 
## Phase 3: Security & Authentication
 
### 3.1 Site Authentication
 
```typescript
// Option A: API Key
const bridge = new McpWebBridge({
  hubUrl: 'wss://hub.example.com',
  apiKey: 'site_xxxxx',
});
 
// Option B: OAuth
const bridge = new McpWebBridge({
  hubUrl: 'wss://hub.example.com',
  auth: {
    type: 'oauth',
    clientId: 'xxx',
    getAccessToken: () => fetchAccessToken(),
  },
});
```
 
### 3.2 Tool Permissions
 
```typescript
bridge.defineTool({
  name: 'deleteAllData',
  description: 'Delete all user data',
  inputSchema: { ... },
  // Permission metadata
  permissions: {
    dangerous: true,
    requiresConfirmation: true,
    scopes: ['data:write', 'data:delete'],
  },
  handler: async () => { ... },
});
```
 
### 3.3 Rate Limiting & Abuse Prevention
 
- Per-site rate limits
- Per-tool rate limits
- Request size limits
- Connection limits per origin
 
---
 
## Phase 4: Production Features
 
### 4.1 Multiple Transport Support
 
```
┌─────────────────────────────────────────────┐
│              MCP Web Bridge                  │
├─────────────────────────────────────────────┤
│  Transports:                                │
│  ├── stdio (Claude Desktop)                 │
│  ├── SSE/HTTP (Web clients)                 │
│  └── WebSocket (Real-time clients)          │
└─────────────────────────────────────────────┘
```
 
### 4.2 Admin Dashboard (`packages/dashboard`)
 
- View connected sites
- Monitor tool usage
- View request logs
- Manage API keys
- Block/allow sites
 
---
 
## Project Structure
 
```
mcp-web-bridge/
├── packages/
│   ├── server/                 # Hub server
│   │   ├── src/
│   │   ├── tests/
│   │   └── package.json
│   ├── client/                 # Browser client SDK
│   │   ├── src/
│   │   ├── tests/
│   │   └── package.json
│   ├── types/                  # Shared TypeScript types
│   │   ├── src/
│   │   └── package.json
│   └── cli/                    # CLI tool
│       ├── src/
│       └── package.json
├── examples/
│   ├── todo-app/
│   ├── ecommerce/
│   ├── calendar/
│   └── minimal/                # Bare minimum example
├── docs/
│   ├── protocol.md             # Protocol specification
│   ├── getting-started.md
│   ├── security.md
│   ├── api-reference.md
│   └── claude-desktop.md       # Claude Desktop setup guide
├── .github/
│   ├── workflows/
│   │   ├── ci.yml              # Test & lint
│   │   ├── release.yml         # npm publish
│   │   └── docs.yml            # Deploy docs
│   ├── ISSUE_TEMPLATE/
│   │   ├── bug_report.md
│   │   └── feature_request.md
│   ├── PULL_REQUEST_TEMPLATE.md
│   └── CONTRIBUTING.md
├── README.md
├── LICENSE                     # MIT
├── package.json                # Workspace root
├── turbo.json                  # Turborepo config
├── tsconfig.json               # Base TS config
├── .eslintrc.js
├── .prettierrc
└── CHANGELOG.md
```
 
---
 
## Testing Strategy
 
### Unit Tests
 
```typescript
// packages/server/tests/registry.test.ts
describe('ToolRegistry', () => {
  it('should namespace tools by origin', () => {
    const registry = new ToolRegistry();
    registry.register('site-1', 'example.com', [
      { name: 'getTodos', description: '...', inputSchema: {} }
    ]);
 
    const tools = registry.getAll();
    expect(tools[0].name).toBe('example.com/getTodos');
  });
 
  it('should handle tool name collisions across sites', () => {
    const registry = new ToolRegistry();
    registry.register('site-1', 'a.com', [{ name: 'getData', ... }]);
    registry.register('site-2', 'b.com', [{ name: 'getData', ... }]);
 
    const tools = registry.getAll();
    expect(tools).toHaveLength(2);
    expect(tools.map(t => t.name)).toContain('a.com/getData');
    expect(tools.map(t => t.name)).toContain('b.com/getData');
  });
 
  it('should remove tools when site disconnects', () => { ... });
});
```
 
### Integration Tests
 
```typescript
// packages/server/tests/integration.test.ts
describe('Hub Integration', () => {
  let hub: Hub;
  let mcpServer: McpServer;
 
  beforeEach(async () => {
    hub = new Hub({ port: 0 }); // Random port
    mcpServer = new McpServer(hub);
    await hub.start();
  });
 
  it('should expose tools from connected site', async () => {
    // Connect mock site
    const client = new WebSocket(hub.url);
    await sendMessage(client, {
      type: 'register',
      tools: [{ name: 'test', description: 'Test tool', inputSchema: {} }]
    });
 
    // Query MCP server
    const tools = await mcpServer.listTools();
    expect(tools).toContainEqual(
      expect.objectContaining({ name: 'localhost/test' })
    );
  });
 
  it('should execute tool on correct site', async () => { ... });
 
  it('should handle site disconnection gracefully', async () => { ... });
});
```
 
### E2E Tests
 
```typescript
// e2e/claude-desktop.test.ts (manual/semi-automated)
describe('Claude Desktop E2E', () => {
  it('should list tools from connected sites', async () => {
    // 1. Start hub server
    // 2. Open example site in puppeteer
    // 3. Verify MCP tools list includes site tools
  });
});
```
 
### Test Commands
 
```json
{
  "scripts": {
    "test": "turbo run test",
    "test:unit": "turbo run test:unit",
    "test:integration": "turbo run test:integration",
    "test:e2e": "playwright test",
    "test:coverage": "turbo run test:coverage"
  }
}
```
 
---
 
## CI/CD Pipeline
 
### `.github/workflows/ci.yml`
 
```yaml
name: CI
 
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
 
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm run lint
      - run: npm run typecheck
 
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm run test:coverage
      - uses: codecov/codecov-action@v4
 
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm run build
```
 
### `.github/workflows/release.yml`
 
```yaml
name: Release
 
on:
  push:
    tags:
      - 'v*'
 
jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          registry-url: 'https://registry.npmjs.org'
      - run: npm ci
      - run: npm run build
      - run: npm run test
      - run: npx changeset publish
        env:
          NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
```
 
---
 
## Documentation Plan
 
### README.md (Root)
 
```markdown
# MCP Web Bridge
 
Connect any website to AI agents through the Model Context Protocol.
 
## What is this?
 
MCP Web Bridge allows websites to expose tools that AI agents (like Claude)
can discover and use. Think of it as a universal adapter between web apps
and AI.
 
## Quick Start
 
### 1. Run the Hub Server
 
\`\`\`bash
npx @mcp-web-bridge/server
\`\`\`
 
### 2. Add to Your Website
 
\`\`\`html
<script src="https://unpkg.com/@mcp-web-bridge/client"></script>
<script>
  const bridge = new McpWebBridge({ hubUrl: 'ws://localhost:9800' });
 
  bridge.defineTool({
    name: 'greet',
    description: 'Say hello to someone',
    inputSchema: {
      type: 'object',
      properties: { name: { type: 'string' } },
      required: ['name']
    },
    handler: async ({ name }) => `Hello, ${name}!`
  });
 
  bridge.connect();
</script>
\`\`\`
 
### 3. Configure Claude Desktop
 
\`\`\`json
{
  "mcpServers": {
    "web-bridge": {
      "command": "npx",
      "args": ["@mcp-web-bridge/server"]
    }
  }
}
\`\`\`
 
### 4. Use It!
 
Ask Claude: "Use the greet tool to say hello to Alice"
 
## Examples
 
- [Todo App](./examples/todo-app) - Simple CRUD example
- [E-commerce](./examples/ecommerce) - Product search & cart
- [Calendar](./examples/calendar) - Event management
 
## Documentation
 
- [Getting Started](./docs/getting-started.md)
- [Protocol Specification](./docs/protocol.md)
- [Security Guide](./docs/security.md)
- [API Reference](./docs/api-reference.md)
 
## Contributing
 
See [CONTRIBUTING.md](./.github/CONTRIBUTING.md)
 
## License
 
MIT
```
 
### docs/getting-started.md
 
1. Installation
2. Running the hub server
3. Integrating client SDK
4. Defining tools
5. Testing with Claude Desktop
6. Deployment options
 
### docs/protocol.md
 
1. Overview
2. Connection lifecycle
3. Message formats
4. Tool namespacing
5. Error handling
6. Versioning
 
### docs/security.md
 
1. Authentication options
2. Origin validation
3. Tool permissions
4. Rate limiting
5. Best practices
 
---
 
## Milestones
 
### v0.1.0 - MVP (Week 1-2)
- [ ] Hub server with WebSocket support
- [ ] Basic MCP server (stdio transport)
- [ ] Client SDK (browser)
- [ ] Tool namespacing
- [ ] Minimal example (todo app)
- [ ] README with quick start
 
### v0.2.0 - Examples & Docs (Week 3)
- [ ] E-commerce example
- [ ] Calendar example
- [ ] Full documentation
- [ ] Claude Desktop setup guide
- [ ] CI/CD pipeline
 
### v0.3.0 - Security (Week 4)
- [ ] Origin validation
- [ ] API key authentication
- [ ] Rate limiting
- [ ] Tool permissions metadata
 
### v0.4.0 - Production Ready (Week 5-6)
- [ ] SSE transport
- [ ] Reconnection handling
- [ ] Admin dashboard
- [ ] npm packages published
- [ ] Docker image
 
### v1.0.0 - Stable Release
- [ ] OAuth support
- [ ] Hosted service option
- [ ] Comprehensive test coverage
- [ ] Security audit
- [ ] Logo & branding
 
---
 
## Technology Choices
 
| Component | Technology | Rationale |
|-----------|------------|-----------|
| Language | TypeScript | Type safety, MCP SDK compatibility |
| Hub Server | Node.js + ws | Simple, proven WebSocket library |
| MCP Server | @modelcontextprotocol/sdk | Official SDK |
| Build | tsup | Fast, simple bundling |
| Monorepo | Turborepo | Fast builds, good DX |
| Testing | Vitest | Fast, ESM-native |
| Linting | ESLint + Prettier | Standard tooling |
| Docs | VitePress | Simple, looks good |
| CI/CD | GitHub Actions | Free for open source |
 
---
 
## Open Questions
 
1. **Naming convention for tools** - `origin.com/toolName` vs `origin.com:toolName` vs `[origin.com]toolName`?
 
2. **Tool update strategy** - Should sites be able to update tools dynamically, or only on reconnect?
 
3. **Multi-tab handling** - If same site is open in multiple tabs, how to handle? (probably: latest connection wins)
 
4. **Offline tools** - Should we cache tool definitions when sites disconnect? (probably: no, tools disappear when site disconnects)
 
5. **Subdomain handling** - Should `app.example.com` and `api.example.com` be treated as same origin?
 
---
 
## Success Metrics
 
- GitHub stars (target: 500 in first month)
- npm weekly downloads (target: 1000 in first month)
- Number of example integrations
- Community contributions
- Documentation completeness
- Test coverage (target: >80%)
 
Update Todos
 Write implementation plan document
 Create plan sections: Overview, Architecture, Phases, File Structure
 Define testing strategy and CI/CD
 Plan examples and documentation
I've written a comprehensive implementation plan at mcp-web-bridge-plan.md. Here's the summary:

Plan Overview
Packages (Monorepo)
packages/
├── server/     # Hub server (WebSocket + MCP)
├── client/     # Browser SDK
├── types/      # Shared types
└── cli/        # CLI tool

Phases
Phase	Focus	Deliverables
1	MVP	Hub server, client SDK, tool namespacing
2	Examples	Todo app, e-commerce, calendar + Claude Desktop guide
3	Security	Auth, rate limiting, permissions
4	Production	SSE transport, admin dashboard, Docker
Examples
Todo App - Simple CRUD (addTodo, listTodos, completeTodo)
E-commerce - Product search, cart, checkout
Calendar - Event CRUD
Minimal - Bare minimum integration
Key Design Decisions
Tool namespacing: origin.com/toolName (prevents collisions)
Transports: stdio (Claude Desktop), SSE (web), WebSocket (real-time)
Monorepo: Turborepo with TypeScript throughout
Testing: Vitest for unit/integration, Playwright for E2E
Open Questions (Need Your Input)
Confirm repo name: mcp-web-bridge?
Tool naming: origin.com/tool vs origin.com:tool?
License preference: MIT?

