# MCP-Web-Bridge Documentation Structure PRD

> Product Requirements Document for implementing Claude Code documentation structure

---

## Executive Summary

This PRD outlines the implementation plan for creating Claude Code-compatible documentation for the MCP-Web-Bridge project. Based on the latest 2025-2026 Anthropic documentation standards, we will implement:

1. **CLAUDE.md** - Project-level instructions
2. **Custom Subagents** - Task delegation with isolated context
3. **Skills** - Specialized knowledge for specific tasks

---

## 2025-2026 Claude Code Documentation Structure

### Directory Structure

```
MCP-Web-Bridge/
├── CLAUDE.md                              # Project instructions (< 300 lines)
├── .claude/
│   ├── agents/                            # Custom subagents
│   │   ├── mcp-protocol-expert.md
│   │   ├── hub-server-developer.md
│   │   ├── client-sdk-developer.md
│   │   ├── security-reviewer.md
│   │   ├── test-writer.md
│   │   └── documentation-writer.md
│   └── skills/
│       ├── hub-server/
│       │   ├── SKILL.md
│       │   └── reference.md
│       ├── client-sdk/
│       │   ├── SKILL.md
│       │   └── examples.md
│       ├── mcp-protocol/
│       │   ├── SKILL.md
│       │   └── message-formats.md
│       ├── websocket-debugging/
│       │   └── SKILL.md
│       └── tool-registry/
│           └── SKILL.md
```

---

## Reference: Latest Documentation Standards

### CLAUDE.md Best Practices (2025-2026)

| Aspect | Recommendation |
|--------|----------------|
| **Length** | < 300 lines (shorter is better, ~60 lines optimal) |
| **Format** | Markdown, human-readable |
| **Content** | Repo etiquette, dev setup, unexpected behaviors |
| **Location** | Root of repo (check into git) |
| **Iteration** | Refine like prompts; use "IMPORTANT", "YOU MUST" |

### Skills Structure (2025-2026)

```markdown
---
name: skill-name                    # Required: lowercase, alphanumeric + hyphens
description: What and when to use   # Required: max 1024 chars
allowed-tools: Tool1, Tool2         # Optional: comma-separated tools
model: claude-sonnet-4-20250514     # Optional: specific model
context: fork                       # Optional: isolated context
agent: general-purpose              # Optional: agent type
user-invocable: true                # Optional: show in slash menu
hooks:                              # Optional: lifecycle hooks
  PreToolUse: [...]
---

# Skill Name

## Instructions
Clear step-by-step guidance

## Examples
Concrete examples
```

### Subagents Structure (2025-2026)

```markdown
---
name: agent-name                    # Required
description: What the agent does    # Required
tools: Read, Glob, Grep             # Optional: whitelist tools
model: sonnet                       # Optional: haiku, sonnet, opus
skills: skill1, skill2              # Optional: accessible skills
---

System prompt for the agent in Markdown
```

### Tool Categories by Agent Role

| Role | Tools |
|------|-------|
| **Read-only** (reviewers) | Read, Grep, Glob |
| **Research** (analysts) | Read, Grep, Glob, WebFetch, WebSearch |
| **Code writers** | Read, Write, Edit, Bash, Glob, Grep |

---

## Session PRDs

Each session below is designed to be completable in a single working session with clear acceptance criteria.

---

## Session 1: CLAUDE.md - Project Instructions

### Objective
Create the root CLAUDE.md file that provides project-wide instructions for all Claude Code sessions.

### Tasks

#### Task 1.1: Create CLAUDE.md structure
**Description:** Create the root CLAUDE.md file with basic structure

**Acceptance Criteria:**
- [ ] File exists at `/home/user/MCP-Web-Bridge/CLAUDE.md`
- [ ] Contains project overview section
- [ ] Under 300 lines total
- [ ] Human-readable markdown format

#### Task 1.2: Project Overview Section
**Description:** Document what MCP-Web-Bridge is and its vision

**Content Requirements:**
- [ ] One-paragraph project description
- [ ] Architecture diagram (ASCII)
- [ ] Link to plan.md for full details

#### Task 1.3: Development Setup Section
**Description:** Document how to set up the development environment

**Content Requirements:**
- [ ] Required Node.js version
- [ ] Package manager (npm/pnpm)
- [ ] How to install dependencies
- [ ] How to run dev server
- [ ] How to run tests

#### Task 1.4: Repository Conventions Section
**Description:** Document coding standards and conventions

**Content Requirements:**
- [ ] Branch naming conventions
- [ ] Commit message format
- [ ] PR process
- [ ] Code style (TypeScript, ESLint, Prettier)

#### Task 1.5: Architecture Guidelines Section
**Description:** Document key architectural decisions

**Content Requirements:**
- [ ] Monorepo structure explanation
- [ ] Package responsibilities
- [ ] Key design patterns used
- [ ] Important: Tool namespacing convention

#### Task 1.6: Common Commands Section
**Description:** List frequently used commands

**Content Requirements:**
- [ ] Build commands
- [ ] Test commands
- [ ] Lint commands
- [ ] Package-specific commands

### Deliverable
Complete CLAUDE.md file under 300 lines covering all sections.

---

## Session 2: Core Subagents - MCP Protocol Expert & Hub Server Developer

### Objective
Create the foundational subagents for MCP protocol knowledge and hub server development.

### Tasks

#### Task 2.1: Create .claude/agents directory
**Description:** Set up the agents directory structure

**Acceptance Criteria:**
- [ ] Directory exists at `.claude/agents/`
- [ ] .gitkeep or initial file present

#### Task 2.2: MCP Protocol Expert Agent
**Description:** Create an agent specialized in MCP protocol knowledge

**File:** `.claude/agents/mcp-protocol-expert.md`

**Content Requirements:**
- [ ] YAML frontmatter with name, description
- [ ] Tools: Read, Grep, Glob, WebFetch, WebSearch
- [ ] Model: sonnet
- [ ] System prompt covering:
  - MCP protocol fundamentals
  - Message format knowledge
  - Tool definition standards
  - Transport mechanisms (stdio, SSE, WebSocket)

**Acceptance Criteria:**
- [ ] Valid YAML frontmatter
- [ ] Clear, actionable system prompt
- [ ] Appropriate tool restrictions

#### Task 2.3: Hub Server Developer Agent
**Description:** Create an agent specialized in hub server development

**File:** `.claude/agents/hub-server-developer.md`

**Content Requirements:**
- [ ] YAML frontmatter with name, description
- [ ] Tools: Read, Write, Edit, Bash, Glob, Grep
- [ ] Model: sonnet
- [ ] Skills: hub-server, mcp-protocol
- [ ] System prompt covering:
  - WebSocket hub implementation
  - Site connection management
  - Tool registry operations
  - MCP server integration

**Acceptance Criteria:**
- [ ] Valid YAML frontmatter
- [ ] Skills field references planned skills
- [ ] Clear development guidelines

### Deliverable
Two functional subagent definition files.

---

## Session 3: Development Subagents - Client SDK & Test Writer

### Objective
Create subagents for client SDK development and test writing.

### Tasks

#### Task 3.1: Client SDK Developer Agent
**Description:** Create an agent specialized in browser client SDK development

**File:** `.claude/agents/client-sdk-developer.md`

**Content Requirements:**
- [ ] YAML frontmatter with name, description
- [ ] Tools: Read, Write, Edit, Bash, Glob, Grep
- [ ] Model: sonnet
- [ ] Skills: client-sdk, mcp-protocol
- [ ] System prompt covering:
  - Browser WebSocket client
  - Tool builder fluent API
  - Reconnection handling
  - TypeScript best practices for browser

**Acceptance Criteria:**
- [ ] Valid YAML frontmatter
- [ ] Browser-specific guidance
- [ ] Clear API design principles

#### Task 3.2: Test Writer Agent
**Description:** Create an agent specialized in writing tests

**File:** `.claude/agents/test-writer.md`

**Content Requirements:**
- [ ] YAML frontmatter with name, description
- [ ] Tools: Read, Write, Edit, Bash, Glob, Grep
- [ ] Model: sonnet
- [ ] System prompt covering:
  - Vitest testing patterns
  - Unit test structure
  - Integration test patterns
  - Mock/stub strategies for WebSocket
  - Test coverage requirements

**Acceptance Criteria:**
- [ ] Valid YAML frontmatter
- [ ] Testing best practices documented
- [ ] Coverage expectations defined

### Deliverable
Two functional subagent definition files for client SDK and testing.

---

## Session 4: Quality Subagents - Security Reviewer & Documentation Writer

### Objective
Create subagents for security review and documentation.

### Tasks

#### Task 4.1: Security Reviewer Agent
**Description:** Create an agent specialized in security review

**File:** `.claude/agents/security-reviewer.md`

**Content Requirements:**
- [ ] YAML frontmatter with name, description
- [ ] Tools: Read, Grep, Glob (read-only)
- [ ] Model: sonnet
- [ ] System prompt covering:
  - WebSocket security concerns
  - Origin validation
  - Input sanitization
  - Rate limiting review
  - Authentication/authorization patterns
  - OWASP considerations

**Acceptance Criteria:**
- [ ] Read-only tool access
- [ ] Security checklist included
- [ ] No write capabilities

#### Task 4.2: Documentation Writer Agent
**Description:** Create an agent specialized in technical documentation

**File:** `.claude/agents/documentation-writer.md`

**Content Requirements:**
- [ ] YAML frontmatter with name, description
- [ ] Tools: Read, Write, Edit, Glob, Grep
- [ ] Model: sonnet
- [ ] System prompt covering:
  - API documentation style
  - README conventions
  - Code example format
  - Markdown best practices

**Acceptance Criteria:**
- [ ] Valid YAML frontmatter
- [ ] Documentation standards defined
- [ ] Example templates referenced

### Deliverable
Two functional subagent definition files for security and documentation.

---

## Session 5: Core Skills - Hub Server & MCP Protocol

### Objective
Create foundational skills for hub server development and MCP protocol knowledge.

### Tasks

#### Task 5.1: Create .claude/skills directory structure
**Description:** Set up the skills directory

**Acceptance Criteria:**
- [ ] Directory exists at `.claude/skills/`
- [ ] Subdirectories for each skill

#### Task 5.2: Hub Server Skill
**Description:** Create skill for hub server development

**Directory:** `.claude/skills/hub-server/`

**Files:**
- `SKILL.md` - Main skill file
- `reference.md` - API reference

**SKILL.md Requirements:**
- [ ] YAML frontmatter:
  - name: hub-server
  - description: Development of WebSocket hub for site connections
  - allowed-tools: Read, Write, Edit, Bash, Glob, Grep
- [ ] Instructions for:
  - Hub class structure
  - Site connection lifecycle
  - Event emission patterns
  - Error handling

**reference.md Requirements:**
- [ ] Hub API reference
- [ ] ConnectedSite interface
- [ ] Event types

**Acceptance Criteria:**
- [ ] Valid YAML frontmatter
- [ ] Under 500 lines for SKILL.md
- [ ] Progressive disclosure with reference.md

#### Task 5.3: MCP Protocol Skill
**Description:** Create skill for MCP protocol knowledge

**Directory:** `.claude/skills/mcp-protocol/`

**Files:**
- `SKILL.md` - Main skill file
- `message-formats.md` - Message type reference

**SKILL.md Requirements:**
- [ ] YAML frontmatter:
  - name: mcp-protocol
  - description: MCP protocol knowledge for tool definitions and message handling
  - allowed-tools: Read, Grep, Glob
- [ ] Instructions for:
  - Tool definition structure
  - Message types
  - Transport options
  - Error responses

**message-formats.md Requirements:**
- [ ] All message type definitions
- [ ] Examples of each message type

**Acceptance Criteria:**
- [ ] Valid YAML frontmatter
- [ ] Comprehensive protocol coverage
- [ ] Clear message examples

### Deliverable
Two complete skill directories with SKILL.md and supporting files.

---

## Session 6: Development Skills - Client SDK & Tool Registry

### Objective
Create skills for client SDK development and tool registry management.

### Tasks

#### Task 6.1: Client SDK Skill
**Description:** Create skill for browser client SDK development

**Directory:** `.claude/skills/client-sdk/`

**Files:**
- `SKILL.md` - Main skill file
- `examples.md` - Usage examples

**SKILL.md Requirements:**
- [ ] YAML frontmatter:
  - name: client-sdk
  - description: Browser client SDK for connecting websites to MCP hub
  - allowed-tools: Read, Write, Edit, Bash, Glob, Grep
- [ ] Instructions for:
  - McpWebBridge class design
  - Tool definition API
  - Connection management
  - Browser compatibility

**examples.md Requirements:**
- [ ] Basic connection example
- [ ] Tool definition examples
- [ ] Error handling examples
- [ ] Reconnection examples

**Acceptance Criteria:**
- [ ] Valid YAML frontmatter
- [ ] Browser-focused instructions
- [ ] Clear code examples

#### Task 6.2: Tool Registry Skill
**Description:** Create skill for tool registry management

**Directory:** `.claude/skills/tool-registry/`

**Files:**
- `SKILL.md` - Main skill file

**SKILL.md Requirements:**
- [ ] YAML frontmatter:
  - name: tool-registry
  - description: Tool registry for aggregating and namespacing tools from connected sites
  - allowed-tools: Read, Write, Edit, Bash, Glob, Grep
- [ ] Instructions for:
  - Tool namespacing (origin.com/toolName)
  - Registration/unregistration
  - Collision handling
  - Resolution logic

**Acceptance Criteria:**
- [ ] Valid YAML frontmatter
- [ ] Clear namespacing rules
- [ ] Edge case handling documented

### Deliverable
Two complete skill directories for client SDK and tool registry.

---

## Session 7: Utility Skill - WebSocket Debugging

### Objective
Create a utility skill for debugging WebSocket connections.

### Tasks

#### Task 7.1: WebSocket Debugging Skill
**Description:** Create skill for debugging WebSocket issues

**Directory:** `.claude/skills/websocket-debugging/`

**Files:**
- `SKILL.md` - Main skill file

**SKILL.md Requirements:**
- [ ] YAML frontmatter:
  - name: websocket-debugging
  - description: Debugging WebSocket connections, message flows, and connection issues
  - allowed-tools: Read, Bash, Grep, Glob
- [ ] Instructions for:
  - Connection troubleshooting
  - Message inspection
  - Common error patterns
  - Testing with wscat/websocat
  - Browser DevTools guidance

**Acceptance Criteria:**
- [ ] Valid YAML frontmatter
- [ ] Practical debugging steps
- [ ] Tool recommendations

### Deliverable
Complete websocket-debugging skill directory.

---

## Session 8: Integration & Validation

### Objective
Validate all documentation files work together correctly.

### Tasks

#### Task 8.1: Validate CLAUDE.md
**Description:** Ensure CLAUDE.md is valid and under line limit

**Acceptance Criteria:**
- [ ] Under 300 lines
- [ ] All links work
- [ ] No typos in commands
- [ ] Consistent formatting

#### Task 8.2: Validate All Agents
**Description:** Ensure all agent files have valid YAML frontmatter

**Acceptance Criteria:**
- [ ] All YAML parses correctly
- [ ] Required fields present (name, description)
- [ ] Skills references match actual skills
- [ ] Tools are valid tool names

#### Task 8.3: Validate All Skills
**Description:** Ensure all skill files have valid YAML frontmatter

**Acceptance Criteria:**
- [ ] All YAML parses correctly
- [ ] Required fields present (name, description)
- [ ] SKILL.md under 500 lines each
- [ ] Supporting files linked correctly

#### Task 8.4: Create Documentation Index
**Description:** Create an index of all documentation files

**File:** `docs/claude-code-setup.md`

**Content Requirements:**
- [ ] List of all agents with descriptions
- [ ] List of all skills with descriptions
- [ ] How to use each component
- [ ] Troubleshooting section

### Deliverable
Validated documentation suite with index file.

---

## Summary

| Session | Focus | Deliverables | Est. Files |
|---------|-------|--------------|------------|
| 1 | CLAUDE.md | Project instructions | 1 |
| 2 | Core Agents | MCP Expert, Hub Developer | 2 |
| 3 | Dev Agents | Client SDK Dev, Test Writer | 2 |
| 4 | Quality Agents | Security, Documentation | 2 |
| 5 | Core Skills | Hub Server, MCP Protocol | 4 |
| 6 | Dev Skills | Client SDK, Tool Registry | 3 |
| 7 | Utility Skills | WebSocket Debugging | 1 |
| 8 | Integration | Validation & Index | 1 |

**Total: 16 files across 8 sessions**

---

## Sources

- [Agent Skills - Claude Code Docs](https://code.claude.com/docs/en/skills)
- [Create custom subagents - Claude Code Docs](https://code.claude.com/docs/en/sub-agents)
- [Claude Code: Best practices for agentic coding - Anthropic](https://www.anthropic.com/engineering/claude-code-best-practices)
- [GitHub - anthropics/skills](https://github.com/anthropics/skills)
- [Building agents with the Claude Agent SDK - Anthropic](https://www.anthropic.com/engineering/building-agents-with-the-claude-agent-sdk)
