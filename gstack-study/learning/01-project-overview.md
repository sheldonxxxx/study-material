# gstack Project Overview

## Purpose

**gstack** is a TypeScript-based CLI toolkit and skill framework for AI-assisted software development. It provides browser automation, code review, planning, and QA capabilities through a modular system of skills that extend Claude Code.

## Scope

### Core Deliverables

1. **Browser Automation Tool** (`browse/`)
   - CLI for controlling Chromium via Playwright
   - Commands for navigation, content extraction, interaction, dialogs
   - Cookie import from browser profiles
   - Screenshot and PDF generation

2. **Skill Framework** (`.agents/skills/`, `mode directories`)
   - 35+ skill modules (e.g., `/review`, `/qa`, `/ship`, `/cso`)
   - Each skill is a self-contained SKILL.md with instructions
   - Skills discovered and loaded by Claude Code automatically

3. **CLI Utilities** (`bin/`, `lib/`)
   - Global skill discovery binary
   - Shared utilities (config, analytics, diff-scope)

### Technology Stack

| Component | Technology |
|-----------|------------|
| Runtime | Bun >= 1.0.0 |
| Language | TypeScript |
| Browser Automation | Playwright (Chromium) |
| Agents | Claude Code, Codex, Gemini CLI |
| Database | Supabase (telemetry) |
| CI | GitHub Actions |

### Directory Structure

```
gstack/
├── .agents/skills/       # 35+ skill definitions
├── agents/               # Agent configurations
├── bin/                  # CLI utilities and binaries
├── browse/               # Main browser automation tool
│   ├── src/              # Source (cli.ts, server.ts, commands)
│   └── test/            # Integration tests
├── lib/                  # Shared utilities
├── scripts/              # Build and evaluation scripts
├── supabase/             # Database functions and migrations
├── test/                 # E2E tests and fixtures
└── [mode dirs]           # Skill implementations (review/, qa/, etc.)
```

### Entry Points

| File | Purpose |
|------|---------|
| `browse/src/cli.ts` | Main `browse` CLI command |
| `browse/src/server.ts` | HTTP server for browser control |
| `bin/gstack-global-discover` | Global skill discovery |
| `SKILL.md` (root) | Main skill definition |
| `CLAUDE.md` | Root instructions for Claude Code |

### Key Characteristics

- **Monorepo**: Single repository with multiple entry points
- **TypeScript-first**: All source code in TypeScript
- **No build step**: Bun runs TypeScript directly
- **Skill-based architecture**: Extensible via SKILL.md files
- **Security-conscious**: Cloud metadata protection, secrets redaction, input validation

### Quality Attributes

| Attribute | Approach |
|-----------|----------|
| Testing | Bun test + Playwright E2E, diff-based selection, gate/periodic tiers |
| Type Safety | TypeScript with interfaces, no visible tsconfig.json |
| Code Style | Convention-based (2-space indent), no ESLint/Prettier |
| Error Handling | Descriptive errors, typed catch blocks, graceful fallbacks |
| Logging | Bounded CircularBuffers (50K entries), structured format |
| Security | SSRF protection, shell injection hardening, parameterized queries |

### Not in Scope

- Web application or API service
- User authentication system (CLI tool)
- Mobile or multi-platform browser support
- Package distribution beyond npm/bun
