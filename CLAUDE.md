# MCP Web Bridge

Open-source protocol enabling websites to expose MCP-compatible tools through a shared hub. AI agents connect once to the hub and gain access to tools from all connected websites.

```
Website A ──┐
Website B ──┼─[WebSocket]─→ MCP Web Bridge Hub ──[MCP]─→ AI Agent (Claude)
Website C ──┘
```

## Project Structure

```
packages/
├── server/     # Hub server (WebSocket + MCP) - Node.js, ws library
├── client/     # Browser SDK - vanilla JS/TS
├── types/      # Shared TypeScript types
├── cli/        # CLI tool
└── dashboard/  # Admin dashboard (Phase 4)

examples/
├── todo-app/   # CRUD operations
├── ecommerce/  # Product search, cart, checkout
├── calendar/   # Event management
└── minimal/    # Bare minimum integration

docs/           # VitePress documentation
```

## Development Setup

```bash
# Requirements: Node.js 20+, npm

# Install dependencies
npm install

# Build all packages
npm run build

# Run tests
npm run test

# Start dev server (when implemented)
npm run dev
```

## Package Scripts

| Command | Description |
|---------|-------------|
| `npm run build` | Build all packages (Turborepo) |
| `npm run test` | Run all tests (Vitest) |
| `npm run test:unit` | Unit tests only |
| `npm run test:integration` | Integration tests |
| `npm run test:coverage` | Tests with coverage |
| `npm run lint` | ESLint check |
| `npm run typecheck` | TypeScript check |

## Architecture

### Core Components

**Hub (`packages/server/src/hub.ts`)**
- Manages WebSocket connections from websites
- Emits events: `site:connected`, `site:disconnected`, `tools:updated`

**Tool Registry (`packages/server/src/registry.ts`)**
- Aggregates tools from all connected sites
- Namespaces tools by origin: `example.com/toolName`
- Handles registration/unregistration

**MCP Server (`packages/server/src/mcp-server.ts`)**
- Exposes aggregated tools via MCP protocol
- Supports transports: stdio (Claude Desktop), SSE (web)

**Client SDK (`packages/client/src/client.ts`)**
- Browser WebSocket client
- Fluent API for tool definition
- Reconnection handling

### Tool Namespacing

IMPORTANT: All tools are namespaced by origin to prevent collisions:
- Site `example.com` registers tool `getTodos` → exposed as `example.com/getTodos`
- Site `other.com` registers tool `getTodos` → exposed as `other.com/getTodos`

### Protocol Messages

**Site → Hub:**
- `register` - Announce available tools
- `update_tools` - Update tool definitions
- `execute_result` - Return execution result

**Hub → Site:**
- `ack` - Connection acknowledgment with site ID
- `execute_request` - Request tool execution

## Code Conventions

### TypeScript
- Strict mode enabled
- Use interfaces for data shapes, types for unions
- Explicit return types on public methods
- No `any` - use `unknown` when type is uncertain

### File Organization
- One class per file
- Export from package `index.ts`
- Tests co-located in `tests/` directory

### Naming
- Files: `kebab-case.ts`
- Classes: `PascalCase`
- Functions/variables: `camelCase`
- Constants: `SCREAMING_SNAKE_CASE`

### Error Handling
- Use custom error classes extending `Error`
- Include error codes for programmatic handling
- Log errors with structured logging (pino)

## Git Conventions

### Branch Naming
- `feat/description` - New features
- `fix/description` - Bug fixes
- `docs/description` - Documentation
- `refactor/description` - Code refactoring

### Commit Messages
Follow conventional commits:
```
type(scope): description

feat(server): add tool namespacing
fix(client): handle reconnection timeout
docs(readme): add quick start guide
```

### Pull Requests
- Reference issue number
- Include test coverage
- Update CHANGELOG.md

## Testing

### Test Structure
```typescript
describe('ComponentName', () => {
  describe('methodName', () => {
    it('should do expected behavior', () => {
      // Arrange
      // Act
      // Assert
    });
  });
});
```

### Mocking WebSocket
Use `ws` library's mock capabilities or `vitest` mocks for WebSocket testing.

## Key Files Reference

| File | Purpose |
|------|---------|
| `packages/server/src/hub.ts` | WebSocket hub for site connections |
| `packages/server/src/registry.ts` | Tool aggregation and namespacing |
| `packages/server/src/mcp-server.ts` | MCP protocol implementation |
| `packages/client/src/client.ts` | Browser WebSocket client |
| `packages/client/src/tool-builder.ts` | Fluent API for tool definition |
| `packages/types/src/index.ts` | Shared TypeScript types |
| `docs/protocol.md` | Protocol specification |

## Security Considerations

IMPORTANT: When implementing:
- Validate all input from WebSocket messages
- Sanitize tool names and arguments
- Implement origin validation
- Add rate limiting per site
- Never trust client-provided data

## Dependencies

| Package | Purpose |
|---------|---------|
| `@modelcontextprotocol/sdk` | Official MCP SDK |
| `ws` | WebSocket server |
| `zod` | Schema validation |
| `pino` | Structured logging |
| `tsup` | TypeScript bundling |
| `vitest` | Testing framework |
| `turborepo` | Monorepo build orchestration |

## Implementation Status

See `plan.md` for full implementation plan with phases and milestones.

Current phase: **Planning** - Setting up Claude Code documentation structure
