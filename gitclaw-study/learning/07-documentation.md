# Documentation

## Overview

GitClaw maintains comprehensive documentation across multiple files, with the README serving as the primary entry point. Documentation is written in Markdown and lives alongside source code in the repository.

## Documentation Files

| File | Purpose | Lines |
|------|---------|-------|
| `README.md` | Primary documentation, quickstart, SDK reference | ~646 |
| `CONTRIBUTING.md` | Contribution guidelines, project structure | ~85 |
| `SOUL.md` | Agent identity and personality | ~17 |
| `RULES.md` | Behavioral constraints for agents | ~8 |
| `LICENSE` | MIT license terms | (single file) |
| `agent.yaml` | Agent manifest documentation | (example in README) |
| `skills/*/SKILL.md` | Skill definitions | (multiple) |

## README.md Structure

### Sections

1. **Why GitClaw** - Conceptual overview of git-native agent design
2. **One-Command Install** - Quick installation via curl
3. **Quick Start** - Basic usage examples
4. **CLI Options** - Full flag reference table
5. **SDK** - Programmatic usage with `query()`, `tool()`, hooks
6. **QueryOptions Reference** - Complete API options table
7. **Message Types** - Event types for SDK streaming
8. **Architecture** - Agent directory structure
9. **Agent Manifest** - `agent.yaml` format specification
10. **Tools** - Built-in and declarative tools
11. **Hooks** - Lifecycle hook system
12. **Skills** - Skill module format
13. **Plugins** - Plugin system (YAML + TypeScript entry)
14. **Multi-Model Support** - Provider configuration
15. **Inheritance & Composition** - Agent extending agents
16. **Compliance & Audit** - Enterprise features
17. **Contributing** - Link to CONTRIBUTING.md

## SDK Documentation

### Core API

```typescript
import { query } from "gitclaw";

// Streaming query
for await (const msg of query({ prompt: "...", dir: "./agent" })) {
  switch (msg.type) {
    case "delta":         // streaming text
    case "assistant":     // complete response
    case "tool_use":      // tool invocation
    case "tool_result":   // tool output
    case "system":        // lifecycle events
  }
}
```

### Custom Tools

```typescript
import { query, tool } from "gitclaw";

const myTool = tool("name", "description", schema, async (args) => {
  return { text: "result", details: {} };
});

for await (const msg of query({ prompt: "...", tools: [myTool] })) {}
```

### Hooks

```typescript
for await (const msg of query({
  prompt: "...",
  hooks: {
    preToolUse: async (ctx) => ({ action: "allow" | "block" | "modify" }),
    onError: async (ctx) => {},
  },
})) {}
```

## Agent Configuration Documentation

### agent.yaml Format

```yaml
spec_version: "0.1.0"
name: my-agent
version: 1.0.0
model:
  preferred: "anthropic:claude-sonnet-4-5-20250929"
  fallback: ["openai:gpt-4o"]
  constraints:
    temperature: 0.7
    max_tokens: 4096
tools: [cli, read, write, memory]
runtime:
  max_turns: 50
  timeout: 120
skills: [code-review, deploy]
```

## SOUL.md

Defines agent personality and identity:

> "You are Gitclaw, a universal git-native agent. You live inside a git repository — your identity, rules, and memory are all files tracked by git."

**Personality traits:**
- Concise
- Competent
- Honest

## RULES.md

Operational rules for agent behavior:

1. Read before modifying
2. No destructive commands without confirmation
3. No secrets in memory
4. Stay in scope
5. Report errors honestly

## CONTRIBUTING.md

### Getting Started

```bash
git clone https://github.com/<your-username>/gitclaw.git
cd gitclaw
npm install
npm run build
npm test
```

### Development Workflow

1. Fork the repository
2. Create feature branch: `git checkout -b feat/my-feature`
3. Make changes in `src/`
4. Build: `npm run build`
5. Test: `npm test`
6. Commit with clear message
7. Push and open PR

### Guidelines

- **TypeScript** - strict mode, all code
- **ESM** - `"type": "module"`
- **Keep it simple** - minimal dependencies
- **Test changes** - add/update tests in `test/`
- **One concern per PR** - focused, reviewable changes

### Commit Message Examples

- `Add voice mode with OpenAI Realtime adapter`
- `Fix memory tool path resolution on Windows`
- `Update SDK query to support abort signals`

## Skill Documentation

Skills are documented via `SKILL.md` files with YAML frontmatter:

```markdown
---
name: example-skill
description: What this skill does
---

# Skill Name

Usage instructions and implementation details.
```

## Plugin Documentation

Comprehensive plugin system documented in README:

- Plugin manifest (`plugin.yaml`) format
- Tool registration
- Hook registration
- Prompt injection
- Memory layers
- Config resolution
- Discovery order (local, global, installed)
- Programmatic API (`GitclawPluginApi`)

## Documentation Gaps

| Area | Status | Notes |
|------|--------|-------|
| API Reference | Partial | README has tables, no dedicated API docs |
| Type Definitions | Incomplete | `dist/exports.d.ts` auto-generated |
| Examples | Limited | Basic SDK examples in README |
| Plugin Gallery | None | No showcase of community plugins |
| Troubleshooting | Minimal | Basic install instructions only |
| Architecture Deep Dive | None | High-level overview only |

## Hosting

- **GitHub**: https://github.com/open-gitagent/gitclaw
- **npm**: https://www.npmjs.com/package/gitclaw
- **Homepage**: https://github.com/open-gitagent/gitclaw (GitHub Pages redirect)
