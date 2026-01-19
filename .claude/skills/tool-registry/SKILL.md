---
name: tool-registry
description: Tool registry for MCP Web Bridge - tool aggregation, namespacing by origin, collision handling, and resolution logic. Use when working on tool registration and lookup code.
allowed-tools: Read, Write, Edit, Bash, Glob, Grep
---

# Tool Registry

This skill provides knowledge for implementing the tool registry that aggregates and namespaces tools from connected sites.

## Core Concept

The tool registry aggregates tools from all connected sites and namespaces them by origin to prevent collisions:

```
Site: example.com           Site: other.com
  └── getTodos                └── getTodos
  └── addTodo                 └── getData

            ↓ Registry ↓

Exposed to MCP:
  └── example.com/getTodos
  └── example.com/addTodo
  └── other.com/getTodos
  └── other.com/getData
```

## Implementation

### Data Structures

```typescript
interface NamespacedTool {
  // Original tool properties
  name: string;              // Namespaced: "origin/toolName"
  description: string;
  inputSchema: JSONSchema;

  // Registry metadata
  siteId: string;            // ID of owning site
  origin: string;            // Site origin (e.g., "example.com")
  originalName: string;      // Original tool name
}

class ToolRegistry {
  // Map: namespacedName -> NamespacedTool
  private tools: Map<string, NamespacedTool> = new Map();

  // Index: siteId -> Set of namespaced names (for efficient unregister)
  private siteTools: Map<string, Set<string>> = new Map();
}
```

### Namespacing Strategy

```typescript
/**
 * Create a namespaced tool name from origin and tool name.
 *
 * Rules:
 * - Use forward slash as separator: origin/toolName
 * - Normalize origin (lowercase, no protocol, no trailing slash)
 * - Preserve original tool name casing
 */
function createNamespacedName(origin: string, toolName: string): string {
  const normalizedOrigin = normalizeOrigin(origin);
  return `${normalizedOrigin}/${toolName}`;
}

function normalizeOrigin(origin: string): string {
  // Remove protocol
  let normalized = origin.replace(/^https?:\/\//, '');
  // Remove trailing slash
  normalized = normalized.replace(/\/$/, '');
  // Lowercase
  return normalized.toLowerCase();
}

// Examples:
// ("https://Example.com/", "getTodos") -> "example.com/getTodos"
// ("http://api.example.com", "getData") -> "api.example.com/getData"
```

### Registration

```typescript
class ToolRegistry {
  register(siteId: string, origin: string, tools: Tool[]): void {
    const normalizedOrigin = normalizeOrigin(origin);

    // Initialize site tool set
    if (!this.siteTools.has(siteId)) {
      this.siteTools.set(siteId, new Set());
    }
    const siteToolSet = this.siteTools.get(siteId)!;

    for (const tool of tools) {
      const namespacedName = createNamespacedName(normalizedOrigin, tool.name);

      // Check for collision from same site (replace)
      if (this.tools.has(namespacedName)) {
        const existing = this.tools.get(namespacedName)!;
        if (existing.siteId !== siteId) {
          // Different site trying to register same namespaced name
          // This shouldn't happen with proper origin validation
          throw new RegistryError(
            `Tool collision: ${namespacedName} already registered by another site`,
            'TOOL_COLLISION'
          );
        }
      }

      const namespacedTool: NamespacedTool = {
        ...tool,
        name: namespacedName,
        siteId,
        origin: normalizedOrigin,
        originalName: tool.name,
      };

      this.tools.set(namespacedName, namespacedTool);
      siteToolSet.add(namespacedName);
    }
  }
}
```

### Unregistration

```typescript
class ToolRegistry {
  unregister(siteId: string): Tool[] {
    const siteToolSet = this.siteTools.get(siteId);
    if (!siteToolSet) return [];

    const removedTools: Tool[] = [];

    for (const namespacedName of siteToolSet) {
      const tool = this.tools.get(namespacedName);
      if (tool) {
        removedTools.push(tool);
        this.tools.delete(namespacedName);
      }
    }

    this.siteTools.delete(siteId);
    return removedTools;
  }
}
```

### Resolution

```typescript
class ToolRegistry {
  /**
   * Resolve a namespaced tool name to site and original tool name.
   * Used when routing tool execution requests.
   */
  resolve(namespacedName: string): { siteId: string; toolName: string } | undefined {
    const tool = this.tools.get(namespacedName);
    if (!tool) return undefined;

    return {
      siteId: tool.siteId,
      toolName: tool.originalName,
    };
  }

  /**
   * Get all registered tools (for MCP tools/list).
   */
  getAll(): NamespacedTool[] {
    return Array.from(this.tools.values());
  }

  /**
   * Get tools for a specific site.
   */
  getForSite(siteId: string): NamespacedTool[] {
    const siteToolSet = this.siteTools.get(siteId);
    if (!siteToolSet) return [];

    return Array.from(siteToolSet)
      .map(name => this.tools.get(name))
      .filter((tool): tool is NamespacedTool => tool !== undefined);
  }
}
```

## Edge Cases

### 1. Same Tool Name, Different Sites

This is the normal case - tools are namespaced and coexist:

```typescript
registry.register('site-1', 'a.com', [{ name: 'getData', ... }]);
registry.register('site-2', 'b.com', [{ name: 'getData', ... }]);

// Result: both "a.com/getData" and "b.com/getData" exist
```

### 2. Same Origin, Multiple Tabs

When same origin connects from multiple tabs, latest wins:

```typescript
// Tab 1 connects
registry.register('site-1', 'example.com', [{ name: 'toolA', ... }]);

// Tab 2 connects with same origin (hub assigns new siteId)
// Hub should disconnect site-1 first, or:
registry.unregister('site-1');
registry.register('site-2', 'example.com', [{ name: 'toolA', name: 'toolB', ... }]);
```

### 3. Tool Update

When a site updates its tools:

```typescript
// Option A: Full replace
registry.unregister(siteId);
registry.register(siteId, origin, newTools);

// Option B: Update in place (preferred)
class ToolRegistry {
  update(siteId: string, origin: string, tools: Tool[]): void {
    this.unregister(siteId);
    this.register(siteId, origin, tools);
  }
}
```

### 4. Invalid Tool Names

Validate tool names before registration:

```typescript
const TOOL_NAME_REGEX = /^[a-zA-Z][a-zA-Z0-9_]*$/;
const MAX_TOOL_NAME_LENGTH = 64;

function validateToolName(name: string): void {
  if (name.length > MAX_TOOL_NAME_LENGTH) {
    throw new ValidationError(`Tool name too long: ${name}`);
  }
  if (!TOOL_NAME_REGEX.test(name)) {
    throw new ValidationError(`Invalid tool name: ${name}`);
  }
}
```

## Performance Considerations

- Use `Map` for O(1) tool lookup
- Maintain site→tools index for efficient unregister
- Consider tool count limits per site (default: 50)
- Consider total tool count limits (default: 1000)
