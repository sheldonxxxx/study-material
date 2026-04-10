# Documentation

## Overview

OpenCode maintains extensive documentation with multi-language support, developer guides, and contributor resources.

## README System

### Main README

Primary documentation at `README.md` with:
- Installation instructions (multiple methods)
- Desktop app information
- Agent system explanation
- Link to full documentation at opencode.ai/docs

### Multi-Language READMEs

22 language translations available:

| Language | File |
|----------|------|
| English | `README.md` |
| Chinese (Simplified) | `README.zh.md` |
| Chinese (Traditional) | `README.zht.md` |
| Korean | `README.ko.md` |
| German | `README.de.md` |
| Spanish | `README.es.md` |
| French | `README.fr.md` |
| Italian | `README.it.md` |
| Danish | `README.da.md` |
| Japanese | `README.ja.md` |
| Polish | `README.pl.md` |
| Russian | `README.ru.md` |
| Bosnian | `README.bs.md` |
| Portuguese (Brasil) | `README.br.md` |
| Arabic | `README.ar.md` |
| Norwegian | `README.no.md` |
| Thai | `README.th.md` |
| Turkish | `README.tr.md` |
| Ukrainian | `README.uk.md` |
| Bengali | `README.bn.md` |
| Greek | `README.gr.md` |
| Vietnamese | `README.vi.md` |

## Key Documentation Files

### CONTRIBUTING.md

Comprehensive contributor guide including:

**Getting Started**:
- Requirements: Bun 1.3+
- Dev setup: `bun install && bun dev`
- Running against different directories

**Core Packages**:
- `packages/opencode` — Core business logic
- `packages/opencode/src/cli/cmd/tui/` — TUI (SolidJS + opentui)
- `packages/app` — Shared web UI (SolidJS)
- `packages/desktop` — Tauri desktop app
- `packages/plugin` — Plugin system

**Development Commands**:
```bash
bun dev              # Local TUI
bun dev serve        # Headless API server (port 4096)
bun dev web          # Web interface
bun dev <directory>  # TUI in specific directory
bun run --cwd packages/desktop tauri dev  # Desktop app
bun run --cwd packages/desktop tauri build  # Build desktop
```

**PR Requirements**:
- Issue-first policy (PRs must reference issues)
- Small, focused PRs
- Conventional commit titles
- No AI-generated walls of text
- Screenshots/videos for UI changes

**Style Guidelines** (from AGENTS.md):
- Single-word names preferred
- Avoid destructuring
- Avoid `else` statements
- Use Bun APIs (e.g., `Bun.file()`)
- Avoid `any` type
- Prefer immutability

### AGENTS.md

Coding style guide enforced for agent-written code:

**Naming**:
- Single-word names by default
- Multi-word only when single word unclear
- Examples: `pid`, `cfg`, `err`, `opts`

**Code Patterns**:
```typescript
// Good
const foo = 1
const journal = await Bun.file(path.join(dir, "journal.json")).json()

// Bad
const fooBar = 1
const journalPath = path.join(dir, "journal.json")
const journal = await Bun.file(journalPath).json()
```

**Control Flow**:
- Avoid `else` statements
- Prefer early returns
- Use `.catch()` over try/catch

**Drizzle Schema**:
- Use snake_case for field names

**Testing**:
- Avoid mocks
- Tests cannot run from repo root
- Run from package directories

### SECURITY.md

Security policy with:

**Threat Model**:
- No sandbox (permission system is UX, not security)
- Server mode opt-in with HTTP Basic Auth
- User controls own config

**Out of Scope**:
- Server access when opted-in
- Sandbox escapes
- LLM provider data handling
- MCP server behavior
- Malicious config files

**Reporting**:
- GitHub Security Advisories
- 6 business day acknowledgement SLA
- Email escalation: security@anoma.ly

## Glossary System

Multi-language glossary at `.opencode/glossary/`:

| Language | File |
|----------|------|
| Arabic | `ar.md` |
| Bosnian | `bs.md` |
| German | `de.md` |
| Danish | `da.md` |
| Spanish | `es.md` |
| French | `fr.md` |
| Japanese | `ja.md` |
| Korean | `ko.md` |
| Norwegian | `no.md` |
| Polish | `pl.md` |
| Russian | `ru.md` |
| Thai | `th.md` |
| Turkish | `tr.md` |
| Chinese (Simplified) | `zh-cn.md` |
| Chinese (Traditional) | `zh-tw.md` |
| Portuguese (Brasil) | `br.md` |

## Package Documentation

### packages/opencode/AGENTS.md

Agent-specific development guide for the core package.

### packages/app/AGENTS.md

Web UI agent guidelines.

### packages/desktop/AGENTS.md

Desktop app agent guidelines.

### packages/desktop-electron/AGENTS.md

Electron-specific agent guidelines.

### github/README.md

GitHub Action integration guide:
- `/opencode explain` — Explain issues
- `/opencode fix` — Fix issues
- PR review capabilities
- Line-specific code review

## Documentation Automation

### docs-locale-sync.yml

GitHub Action that synchronizes README translations.

### docs-update.yml

Documentation update automation.

## Online Documentation

Full documentation at https://opencode.ai/docs covering:
- Agent system
- Configuration
- Provider setup
- Advanced usage
