# gstack Repository Topology

## Overview

**Repository:** `/Users/sheldon/Documents/claw/reference/gstack`
**Type:** Node.js/Bun monorepo with skill-based CLI tools
**Purpose:** AI-assisted software development toolkit (browser automation, code review, planning, QA)

## Directory Structure

```
gstack/
├── .agents/skills/           # Skill definitions (SKILL.md files)
│   ├── gstack/               # Main skill
│   ├── gstack-autoplan/
│   ├── gstack-benchmark/
│   ├── gstack-browse/        # Browser automation skill
│   ├── gstack-canary/
│   ├── gstack-careful/
│   ├── gstack-cso/
│   ├── gstack-design-consultation/
│   ├── gstack-design-review/
│   ├── gstack-document-release/
│   ├── gstack-freeze/
│   ├── gstack-guard/
│   ├── gstack-investigate/
│   ├── gstack-land-and-deploy/
│   ├── gstack-office-hours/
│   ├── gstack-plan-ceo-review/
│   ├── gstack-plan-design-review/
│   ├── gstack-plan-eng-review/
│   ├── gstack-qa/
│   ├── gstack-qa-only/
│   ├── gstack-retro/
│   ├── gstack-review/
│   ├── gstack-setup-browser-cookies/
│   ├── gstack-setup-deploy/
│   ├── gstack-ship/
│   ├── gstack-unfreeze/
│   └── gstack-upgrade/
├── agents/                   # Agent configuration
│   └── openai.yaml
├── bin/                      # Compiled/executable utilities
│   ├── gstack-global-discover       # Global skill discovery binary
│   ├── gstack-global-discover.ts    # Source
│   ├── worktree.ts
│   └── [gstack-* scripts]
├── browse/                   # Browser automation tool (main deliverable)
│   ├── bin/
│   ├── dist/                 # Compiled output
│   ├── scripts/
│   ├── src/                 # Source code
│   │   ├── cli.ts           # CLI entry point
│   │   ├── server.ts       # HTTP server
│   │   ├── commands.ts
│   │   ├── read-commands.ts
│   │   ├── write-commands.ts
│   │   ├── cookie-import-browser.ts
│   │   ├── cookie-picker-ui.ts
│   │   └── snapshot.ts
│   └── test/
├── lib/                      # Shared utilities
│   ├── dev-setup
│   ├── dev-teardown
│   ├── gstack-analytics
│   ├── gstack-config
│   ├── gstack-diff-scope
│   └── [other gstack-* utilities]
├── scripts/                  # Build/utility scripts
│   ├── resolvers/
│   ├── gen-skill-docs.ts
│   ├── skill-check.ts
│   ├── eval-*.ts            # Evaluation scripts
│   └── analytics.ts
├── supabase/                 # Supabase configuration
│   ├── functions/
│   └── migrations/
├── test/                      # Test fixtures and helpers
│   ├── fixtures/
│   └── helpers/
├── .github/
│   ├── docker/
│   └── workflows/
├── docs/
│   └── images/
├── qa/
│   ├── references/
│   └── templates/
└── [mode directories]        # Each is a skill implementation
    ├── agents/               # (see .agents/skills/)
    ├── autoplan/
    ├── benchmark/
    ├── browse/              # Duplicated skill impl
    ├── canary/
    ├── careful/
    ├── codex/
    ├── cso/
    ├── design-consultation/
    ├── design-review/
    ├── document-release/
    ├── freeze/
    ├── gstack-upgrade/
    ├── guard/
    ├── investigate/
    ├── land-and-deploy/
    ├── office-hours/
    ├── plan-ceo-review/
    ├── plan-design-review/
    ├── plan-eng-review/
    ├── qa/
    ├── qa-only/
    ├── retro/
    ├── review/
    ├── setup-browser-cookies/
    ├── setup-deploy/
    ├── ship/
    └── unfreeze/
```

## Entry Points

### Primary Entry Points

| File | Purpose |
|------|---------|
| `browse/src/cli.ts` | Main CLI entry (`browse` command) |
| `browse/src/server.ts` | HTTP server for browser control |
| `setup` | Installation script (builds binary, registers skills) |
| `bin/gstack-global-discover` | Global skill discovery binary |

### Skills Entry Points

Each skill directory contains a `SKILL.md` that defines:
- The slash command (e.g., `/review`, `/qa`, `/ship`)
- Instructions for Claude Code
- What the skill does and when to invoke it

Skills are discovered from `.agents/skills/` and loaded by Claude Code automatically.

### Build Entry Points

| File | Purpose |
|------|---------|
| `package.json` | npm/bun entry (defines `browse` bin) |
| `scripts/gen-skill-docs.ts` | Generates skill documentation |

## Key Files

| File | Purpose |
|------|---------|
| `CLAUDE.md` | Root instructions for Claude Code |
| `SKILL.md` | Main skill definition |
| `ARCHITECTURE.md` | Technical architecture |
| `README.md` | Project documentation |
| `package.json` | Node.js manifest (Bun-based) |
| `VERSION` | Version file |

## Technology Stack

- **Runtime:** Bun >= 1.0.0
- **Language:** TypeScript
- **Browser Automation:** Playwright
- **Agents:** Claude Code, Codex, Gemini CLI (via SKILL.md standard)
