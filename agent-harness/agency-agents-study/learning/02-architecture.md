# agency-agents Architecture

## Overview

**Repository Type:** AI Agent Definition Knowledge Base
**Architecture Pattern:** Content-Driven Multi-Format Publishing System
**Core Principle:** Canonical single-source agent definitions published to multiple AI coding tool formats

This is not a traditional software codebase. The architecture is a **content publishing system** where the product is agent definitions (markdown files) and the build output is tool-specific integration formats.

---

## Architectural Decisions

### AD-01: Domain-Based Organization Over Function-Based

**Decision:** Agents are organized by business domain (engineering, marketing, design) rather than technical function.

**Rationale:** Domain organization makes intuitive sense for human discovery. Non-technical stakeholders (product managers, marketers) can navigate the repository without understanding software architecture concepts.

**Evidence:** 13 top-level domain directories in the repository root, each containing domain-specific agent definitions.

### AD-02: Markdown with YAML Frontmatter as Canonical Format

**Decision:** All agent definitions use markdown with YAML frontmatter for structured metadata.

**Rationale:**
- Human-readable and writable (markdown body)
- Machine-parseable metadata (YAML frontmatter)
- No database or build step required to author agents
- Diff-friendly for version control

**Schema:**
```yaml
---
name: Agent Name
description: One-line description
color: colorname or "#hexcode"
emoji: 🎯
vibe: One-line personality hook
services:                    # optional
  - name: Service Name
    url: https://...
    tier: free|freemium|paid
---
```

### AD-03: Static Definitions, No Runtime Execution

**Decision:** Agents are static definitions with no execution model or runtime state.

**Rationale:** The repository serves as a knowledge base and definition library. Execution belongs to the AI coding tools that consume these definitions.

**Implication:** No agent-to-agent communication is enforced. Workflows are documented patterns, not implemented systems.

### AD-04: Gitignored Generated Output

**Decision:** The `integrations/` directory is gitignored. Users run `convert.sh` locally to generate tool-specific formats.

**Rationale:**
- Prevents merge conflicts from generated files
- Keeps repository lean (only canonical sources versioned)
- Allows users to generate only the tools they use

---

## Module Responsibilities

### Domain Directories (`[domain]/*.md`)

**Responsibility:** Canonical agent definitions — the source of truth

**Contents:**
- 193 markdown files across 13 domains
- Each file: YAML frontmatter + markdown body
- Structured sections: Identity, Mission, Deliverables, Metrics

**Boundaries:** No cross-domain references. Each agent is self-contained.

### `scripts/convert.sh` — Format Conversion Engine

**Responsibility:** Transform canonical markdown into tool-specific formats

**Key operations:**
- Frontmatter extraction (`get_field`, `get_body`)
- Identifier generation (`slugify`)
- Color normalization (named colors -> hex)
- Section splitting (OpenClaw format)
- Accumulation (Aider/Windsurf single-file outputs)

**Boundaries:** Reads domain/*.md, writes to integrations/*/

### `scripts/install.sh` — User Configuration Installer

**Responsibility:** Copy generated agents to user tool configuration directories

**Key operations:**
- Detect supported AI coding tools
- Interactive prompts for installation choices
- Copy files to tool-specific config locations

**Boundaries:** Reads from integrations/*/, writes to user home directory

### `scripts/lint-agents.sh` — Schema Validation

**Responsibility:** Validate agent markdown files for consistency

**Checks:**
- Required frontmatter fields present
- Required sections exist
- No broken internal links

**Boundaries:** Reads domain/*.md only, no writes

### `integrations/` — Generated Output Directory

**Responsibility:** Tool-specific agent configurations (generated, not versioned)

**Tool formats:**
| Tool | Format |
|------|--------|
| Claude Code | Native `.md` + frontmatter |
| Cursor | `.mdc` rules |
| Aider | Single `CONVENTIONS.md` |
| Windsurf | Single `.windsurfrules` |
| OpenClaw | `SOUL.md` + `AGENTS.md` + `IDENTITY.md` |
| OpenCode | `.md` agents |
| Gemini CLI | `skills/*.md` |
| Antigravity | `SKILL.md` per agent |
| Qwen | SubAgent `.md` files |

### `examples/` — Workflow Demonstrations

**Responsibility:** Document multi-agent coordination patterns

**Contents:** Markdown files showing sequential handoffs, parallel activation, quality gates

---

## Communication Patterns

### Build Pipeline Flow

```
domain/*.md (source)
     |
     v
convert.sh (bash pipeline)
     |
     +---> integrations/claude-code/
     +---> integrations/cursor/
     +---> integrations/aider/
     +---> integrations/windsurf/
     +---> integrations/openclaw/
     +---> integrations/opencode/
     +---> integrations/gemini-cli/
     +---> integrations/antigravity/
     +---> integrations/qwen/
     +---> integrations/mcp-memory/
     |
     v
install.sh (user config)
     |
     v
~/.config/... (user machines)
```

### CI Pipeline

```
Pull Request
     |
     v
.github/workflows/lint-agents.yml
     |
     v
lint-agents.sh (validates agent files)
     |
     +---> Pass: merge allowed
     +---> Fail: PR blocked
```

### No Runtime Communication

Agents are static definitions. Any "communication" between agents is:
- Documented in workflow examples (examples/*.md)
- Manual copy-paste of outputs between agents
- Not enforced by any system

---

## File Ownership

| Path Pattern | Purpose | Versioned |
|-------------|---------|-----------|
| `domain/*.md` | Canonical agent definitions | Yes |
| `scripts/*.sh` | Build/infrastructure tooling | Yes |
| `integrations/*/` | Generated output | No (gitignored) |
| `examples/*.md` | Workflow documentation | Yes |
| `.github/workflows/*` | CI configuration | Yes |

---

## Extension Points

### Adding a New Tool

1. Add `convert_<tool>()` function to `convert.sh`
2. Add case entry in main conversion loop
3. Create `integrations/<tool>/` directory structure

**Example pattern:**
```bash
convert_tool() {
  local agent="$1"
  local output_dir="$2"
  # Transform and write
}
```

### Adding a New Domain

1. Create `newdomain/` directory
2. Add to `AGENT_DIRS` array in `convert.sh`
3. Populate with agent `.md` files

### Adding Agent Properties

1. Add field to frontmatter schema
2. Update `lint-agents.sh` to validate
3. Update relevant `convert_<tool>()` functions

---

## Architectural Constraints

| Constraint | Impact |
|------------|--------|
| No runtime state | Agents cannot maintain session or learn |
| No enforced workflows | Coordination is documentation, not implementation |
| Manual installation | Users must run convert + install locally |
| Gitignored output | Generated files not shared via repo |
| Static definitions | No dynamic agent spawning or modification |
