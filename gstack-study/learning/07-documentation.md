# Documentation

## Overview

gstack maintains comprehensive documentation across multiple files. The documentation is tightly integrated with the codebase — SKILL.md files are generated from templates to prevent drift, and the CLAUDE.md serves as a living developer guide.

## Documentation Structure

### Top-Level Docs

| File | Purpose | Audience |
|------|---------|----------|
| `README.md` | Project introduction, quick start, skill catalog | All users |
| `ARCHITECTURE.md` | Design decisions, system internals, daemon model | Contributors |
| `BROWSER.md` | Full command reference for `/browse` skill | Power users |
| `CONTRIBUTING.md` | Dev setup, testing, contributor workflow, PR process | Contributors |
| `ETHOS.md` | Builder philosophy: Boil the Lake, Search Before Building | Anyone interested in the philosophy |
| `CLAUDE.md` | Project-specific dev commands and conventions | Developers working in the repo |
| `CHANGELOG.md` | Release notes for every version | Users tracking updates |
| `SKILL.md` | Auto-generated from template — lists all skills | Claude Code (consumed by the AI) |

### Skills Documentation

Each skill has its own directory with a `SKILL.md` (auto-generated from `SKILL.md.tmpl`):

```
skill/SKILL.md       # Generated — do not edit directly
skill/SKILL.md.tmpl  # Template — edit this, run gen:skill-docs
```

**Generated content includes:**
- Command reference tables (from `commands.ts`)
- Snapshot flag documentation (from `snapshot.ts`)

This eliminates the class of bugs where documentation references non-existent commands.

### Template Documentation

`docs/skills.md` — Deep dives with examples and philosophy for every skill (including Greptile integration).

## Documentation Quality Assessment

### Strengths

1. **Single source of truth for commands** — `commands.ts` is the command registry; docs are generated from it
2. **Template system prevents drift** — SKILL.md files are committed artifacts, regenerated on every build
3. **CLAUDE.md is project-aware** — developers working in the repo have context without reading multiple files
4. **Contributor workflow documented** — `CONTRIBUTING.md` covers dev mode, symlink setup, Conductor workspaces
5. **Eval observability documented** — comprehensive coverage of eval infrastructure, tools, and interpretation
6. **SKILL.md workflow documented** — clear instructions on editing templates vs generated files

### Gaps

1. **No API documentation** — gstack has no HTTP API (CLI-only). This is intentional but not explicitly stated.
2. **No architecture decision records (ADRs)** — ARCHITECTURE.md explains what and why, but design decisions aren't tracked with context
3. **CHANGELOG is large (113KB)** — the changelog has grown to 113K characters. Consider splitting by version range.

## Discoverability

**For new users:**
- README.md quick start (5 steps, 30-second install)
- README.md skill catalog with descriptions

**For contributors:**
- CONTRIBUTING.md is the entry point
- CLAUDE.md has day-to-day dev commands
- CONTRIBUTING.md covers dev mode, testing tiers, dual-host development

**For power users:**
- BROWSER.md is the command reference
- `docs/skills.md` has deep dives

**For Claude Code (the AI agent):**
- `SKILL.md` is consumed directly by the agent
- `CLAUDE.md` provides project-specific context

## Documentation Maintenance

### SKILL.md Freshness

A GitHub Action (`.github/workflows/skill-docs.yml`) runs `gen-skill-docs --dry-run` on every push/PR. If generated files differ from committed, CI fails.

### CHANGELOG Maintenance

CHANGELOG entries are written at `/ship` time by the shipping workflow. The entry describes what THIS branch adds vs the base branch — never fold new work into an existing released entry.

**CHANGELOG style:**
- Lead with what users can now **do** that they couldn't before
- Plain language, no implementation details
- Separate "For contributors" section at the bottom for internal changes

## Doc-to-Code Ratio

| Metric | Value |
|--------|-------|
| Total repo size (excluding .git) | ~21MB |
| Markdown docs | ~1MB |
| Doc-to-code ratio | ~5% |
| CHANGELOG.md alone | 113KB |
| SKILL.md (generated) | ~17KB |

## Integration with CI

CI enforces SKILL.md freshness. There is no automated spell-check, link-check, or style linting for documentation.

## Telemetry Documentation

The README has a dedicated "Privacy & Telemetry" section explaining:
- Opt-in only (default off)
- What's collected: skill name, duration, success/fail, gstack version, OS
- What's never collected: code, file paths, repo names, branch names, prompts
- Local analytics available via `gstack-analytics`
- Supabase schema is public (`supabase/migrations/`)

This is unusually transparent for an open source project.
