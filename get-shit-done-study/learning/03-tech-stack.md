# Tech Stack Analysis

## Language and Runtime

**Primary Language:** JavaScript (CommonJS)
- **Node.js Version:** >= 20.0.0 (per package.json `engines`)
- **Test Matrix:** Node 22 (LTS), Node 24 (Current) on Ubuntu, macOS, Windows
- **Forward Compatibility:** Node 26 compatible

The project uses **CommonJS exclusively** (`require()`, `.cjs` files) rather than ESM (`import`). This is a deliberate architectural choice emphasizing broad Node.js compatibility.

## Core Architecture

### No External Dependencies in Core

The `gsd-tools.cjs` and all library files in `get-shit-done/bin/lib/` use **only Node.js built-ins**. This is a foundational principle:

- `fs`, `path`, `child_process`, `crypto` (Node.js built-ins)
- No third-party runtime dependencies
- Security-critical: minimizes attack surface

### Dev Dependencies Only

```json
"devDependencies": {
  "c8": "^11.0.0",      // Code coverage
  "esbuild": "^0.24.0"   // Hooks bundler
}
```

## Key Libraries

| Library | Version | Purpose |
|---------|---------|---------|
| `c8` | ^11.0.0 | Coverage reporting for tests |
| `esbuild` | ^0.24.0 | Bundling pre-commit hooks |

## Build System

- **Package Manager:** npm (with `npm ci` in CI)
- **Build Script:** `npm run build:hooks` executes `node scripts/build-hooks.js`
- **Hook Compilation:** esbuild bundles `hooks/*.js` into `hooks/dist/`
- **Prepublish:** `prepublishOnly` runs `build:hooks` before npm publish

## Project Structure

```
get-shit-done/
  bin/
    install.js           # Multi-runtime installer (Node.js)
  get-shit-done/
    bin/lib/             # Core library modules (.cjs) — NO external deps
      commands.cjs
      config.cjs
      core.cjs
      frontmatter.cjs
      init.cjs
      milestone.cjs
      model-profiles.cjs
      phase.cjs
      profile-output.cjs
      profile-pipeline.cjs
      roadmap.cjs
      security.cjs
      state.cjs
      template.cjs
      uat.cjs
      verify.cjs
      workstream.cjs
    workflows/           # Workflow definitions (.md)
    references/          # Reference documentation (.md)
    templates/           # File templates
  agents/                # Agent definitions (.md) — 15 specialized agents
  commands/gsd/          # Slash command definitions (.md)
  hooks/
    commands.cjs         # Hook handlers
    gsd-context-monitor.js
    gsd-prompt-guard.js
    gsd-workflow-guard.js
    dist/                # Compiled hooks output
  scripts/
    build-hooks.js      # esbuild bundler
    run-tests.cjs        # Test runner
    prompt-injection-scan.sh
    base64-scan.sh
    secret-scan.sh
  tests/                 # Test files (.test.cjs) — 50+ test suites
```

## Testing Stack

- **Test Runner:** Node.js built-in `node:test`
- **Assertion Library:** Node.js built-in `node:assert/strict`
- **Coverage:** c8 with 70% line coverage requirement
- **Forbidden:** Jest, Mocha, Chai, or any external test framework

## Multi-Runtime Support

GSD is designed to work with multiple AI coding runtimes:

| Runtime | Installation Path | Notes |
|---------|------------------|-------|
| Claude Code | `~/.claude/` | Primary platform |
| OpenCode | `~/.config/opencode/` | Open source adaptation |
| Gemini CLI | `~/.gemini/` | Google CLI |
| Codex | `~/.codex/` | Skills-based installation |
| Copilot | `~/.github/` | GitHub CLI |
| Cursor | `~/.cursor/` | VS Code-based |
| Windsurf | `~/.windsurf/` | Codeium-based |
| Antigravity | `~/.gemini/antigravity/` | Gemini-based |

## Security Architecture

The project has extensive built-in security:

- **Path traversal prevention** via `security.cjs`
- **Prompt injection detection** in `gsd-prompt-guard.js`
- **Shell argument validation** (uses `execFileSync` over `execSync`)
- **Secret scanning** via custom scripts
- **No `${{ }}` in GitHub Actions** — bind to `env:` mappings first

## Why This Stack?

1. **CommonJS over ESM:** Broader compatibility, simpler module resolution
2. **No runtime deps:** Security surface reduction, reliability
3. **Built-in test runner:** Zero external dependencies, stable API
4. **Node.js native:** No transpilation step, direct execution
5. **Multi-runtime:** GSD is runtime-agnostic by design

This stack prioritizes **reliability, security, and zero-dependency reliability** over feature richness.
