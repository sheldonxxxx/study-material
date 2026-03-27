# Tech Stack Analysis: gstack

## Overview

**gstack** is a TypeScript-based CLI tool and skill framework for Claude Code that provides AI-assisted engineering workflows (code review, QA, deployment, etc.). It uses headless browser automation via Playwright and integrates with multiple LLM providers.

## Language & Runtime

| Aspect | Choice | Version |
|--------|--------|---------|
| Primary Language | TypeScript | - |
| Runtime | Bun | >= 1.0.0 |
| Node.js (for Claude CLI) | Node.js | 22 LTS |
| Package Manager | Bun | - |

**Notes:**
- Bun is the primary runtime and package manager
- Node.js 22 LTS is installed separately in the Docker image specifically for running the `@anthropic-ai/claude-code` CLI
- The project is an ES module (`"type": "module"` in package.json)

## Dependencies

### Production Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| playwright | ^1.58.2 | Headless browser automation |
| diff | ^7.0.0 | Text diffing utility |

### Development Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| @anthropic-ai/sdk | ^0.78.0 | Anthropic API client |

## Build System

- **Build Tool**: Bun
- **Build Command**: `bun run build`
- **Compilation**: `bun build --compile` for CLI binaries
- **Documentation Generation**: Custom `gen-skill-docs.ts` script generates SKILL.md files from templates
- **Output**: Compiled binaries in `browse/dist/` (main binary: `browse`)

**Build artifacts:**
```
browse/dist/browse        # Main CLI binary (~58MB, Mach-O arm64)
browse/dist/find-browse  # Helper binary
bin/gstack-global-discover
```

## Testing

| Test Type | Framework | Notes |
|-----------|-----------|-------|
| Unit/Integration | Bun test | Free tests, <2s |
| E2E Browser | Playwright | Chromium-based |
| LLM Evals | Claude -p + LLM judge | Paid, diff-based |

**Test Commands:**
- `bun test` - Free tests only
- `bun run test:evals` - Paid evals (LLM judge + E2E, ~$4/run)
- `bun run test:gate` - Gate-tier tests only (blocks merge)
- `bun run test:periodic` - Weekly cron tests

## Browser Automation

- **Library**: Playwright (Chromium)
- **Playwright Version**: 1.58.2
- **Browser Path**: `/opt/playwright-browsers` (shared, pre-installed in Docker)
- **E2E Tests**: Multiple test suites in `test/` directory (`skill-e2e-*.test.ts`)

## External Integrations

| Service | Purpose | Auth |
|---------|---------|------|
| Anthropic Claude API | LLM processing, E2E via `claude -p` | `ANTHROPIC_API_KEY` |
| OpenAI Codex CLI | Multi-AI second opinion | `~/.codex/` config |
| Google Gemini | Additional LLM evaluation | `GEMINI_API_KEY` |

## CI/CD

**Platform**: GitHub Actions

**Workflows:**
| Workflow | Trigger | Purpose |
|----------|---------|---------|
| `evals.yml` | PR, manual | E2E eval suite (12 parallel jobs) |
| `evals-periodic.yml` | Weekly cron | Periodic-tier tests |
| `ci-image.yml` | Weekly + Dockerfile changes | Build Docker image |
| `skill-docs.yml` | Push | Skill documentation generation |
| `actionlint.yml` | Push/PR | Workflow linting |

**Runner**: Ubicloud (ubicloud-standard-2, ubicloud-standard-8 for heavy E2E)

## Containerization

**Dockerfile**: `.github/docker/Dockerfile.ci`

**Base Image**: Ubuntu 24.04

**Key installed components:**
- Git
- curl, unzip, jq, bc, gpg
- GitHub CLI (`gh`)
- Node.js 22 LTS
- Bun
- Claude CLI (`@anthropic-ai/claude-code`)
- Playwright + Chromium

**Caching strategy:**
- Pre-installed `node_modules` cached at `/opt/node_modules_cache`
- Pre-installed Playwright browsers at `/opt/playwright-browsers`
- Runtime restore via symlink (~0s vs ~15s install)

## Project Structure

```
gstack/
├── browse/           # Headless browser CLI (Playwright-based)
│   ├── src/          # CLI source + commands
│   └── dist/         # Compiled binaries
├── scripts/          # Build + DX tooling (gen-skill-docs, skill-check, etc.)
├── test/             # Skill validation + eval tests
├── bin/              # CLI utilities
├── .github/
│   ├── workflows/    # CI/CD configs
│   └── docker/       # Dockerfile.ci
├── agents/           # Agent configurations
└── package.json      # Root package (ES module)
```

## Monorepo Tooling

**None.** This is a single-package TypeScript project with no monorepo tooling (no Nx, Turborepo, Lerna, pnpm workspaces).

## Platform Support

- **Primary**: macOS (ARM64) - binaries in `browse/dist/` are Mach-O arm64
- **Build from source**: Linux, macOS (Intel), Windows (via Bun)
- **CI Runtime**: Linux (Ubuntu 24.04 via Docker)

## Key Files

| File | Purpose |
|------|---------|
| `package.json` | Dependencies, scripts, engine requirements |
| `.github/docker/Dockerfile.ci` | Pre-baked CI image with all toolchain |
| `browse/src/cli.ts` | Main CLI entry point |
| `setup` | Installation script (builds binary + symlinks) |
