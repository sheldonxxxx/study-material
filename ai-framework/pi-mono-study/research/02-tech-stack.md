# pi-mono Tech Stack Analysis

## Project Overview

**pi-mono** is a monorepo for building AI agents and managing LLM deployments. The primary product is `pi`, an interactive coding agent CLI with read, bash, edit, and write tools.

- **Repository:** `badlogic/pi-mono` (GitHub)
- **License:** MIT
- **Node.js Requirement:** `>=20.0.0`
- **Type:** npm workspaces monorepo (no Turborepo, Nx, or Lerna)

---

## Languages and Runtimes

| Component | Technology | Version/Notes |
|-----------|------------|---------------|
| Primary Language | TypeScript | 5.9.2 (root), 5.7.3 (packages) |
| Target | ES2022 | Module system: Node16 |
| Runtime | Node.js | >=20.0.0 (CI uses Node 22) |
| Binary Builder | Bun | 1.2.20 (for cross-platform binary compilation) |
| Scripting | tsx | 4.20.3 (for running TypeScript scripts) |

---

## Monorepo Tooling

### npm Workspaces

The project uses native npm workspaces for monorepo management (no Turborepo, Nx, or Lerna).

```json
// package.json workspaces configuration
"workspaces": [
  "packages/*",
  "packages/web-ui/example",
  "packages/coding-agent/examples/extensions/with-deps",
  "packages/coding-agent/examples/extensions/custom-provider-anthropic",
  "packages/coding-agent/examples/extensions/custom-provider-gitlab-duo",
  "packages/coding-agent/examples/extensions/custom-provider-qwen-cli"
]
```

### TypeScript Configuration

- **Base config:** `tsconfig.base.json` (shared across all packages)
- **Package configs:** Each package has its own `tsconfig.build.json` extending the base
- **Compiler:** `tsgo` (tsx-based TypeScript compiler wrapper)
- **Module resolution:** Node16
- **Decorator support:** Enabled (`experimentalDecorators`, `emitDecoratorMetadata`)

### Code Quality Tools

| Tool | Purpose | Config |
|------|---------|--------|
| **Biome** | Linting + Formatting | `biome.json` (2.3.5) |
| **TypeScript** | Type checking | `tsc --noEmit` |
| **Husky** | Git hooks | `prepare` script runs husky install |

---

## Packages in Monorepo

### 1. @mariozechner/pi-ai (0.62.0)
**Purpose:** Unified multi-provider LLM API

Provider support:
- OpenAI (Responses API, Completions API, Codex)
- Anthropic
- Google (Gemini, Vertex, Gemini CLI)
- Mistral
- Azure OpenAI (Responses)
- AWS Bedrock

Key dependencies:
- `@anthropic-ai/sdk`: ^0.73.0
- `@google/genai`: ^1.40.0
- `@mistralai/mistralai`: 1.14.1
- `openai`: 6.26.0
- `@aws-sdk/client-bedrock-runtime`: ^3.983.0
- `@sinclair/typebox`: ^0.34.41 (validation)
- `partial-json`: ^0.1.7 (streaming JSON parsing)

### 2. @mariozechner/pi-agent-core (0.62.0)
**Purpose:** General-purpose agent runtime with transport abstraction, state management, and attachment support

Dependencies:
- `@mariozechner/pi-ai`: ^0.62.0
- Testing: `vitest` ^3.2.4

### 3. @mariozechner/pi-coding-agent (0.62.0)
**Purpose:** Interactive coding agent CLI (the main `pi` product)

Key features:
- Read, bash, edit, write tools
- Session management
- Export to HTML
- Theme support
- Binary builds for macOS, Linux, Windows

Key dependencies:
- `@mariozechner/pi-agent-core`: ^0.62.0
- `@mariozechner/pi-ai`: ^0.62.0
- `@mariozechner/pi-tui`: ^0.62.0
- `@silvia-odwyer/photon-node`: ^0.3.4 (image processing)
- `cli-highlight`: ^2.1.11 (syntax highlighting)
- `diff`: ^8.0.2
- `marked`: ^15.0.12 (markdown rendering)
- `yaml`: ^2.8.2
- `undici`: ^7.19.1 (HTTP client)
- `glob`: ^13.0.1

CLI: `pi` (dist/cli.js)

### 4. @mariozechner/pi-mom (0.62.0)
**Purpose:** Slack bot that delegates messages to the pi coding agent

Key dependencies:
- `@slack/socket-mode`: ^2.0.0
- `@slack/web-api`: ^7.0.0
- `@anthropic-ai/sandbox-runtime`: ^0.0.16
- `croner`: ^9.1.0 (cron parsing)

CLI: `mom` (dist/main.js)

### 5. @mariozechner/pi-tui (0.62.0)
**Purpose:** Terminal UI library with differential rendering

Key dependencies:
- `chalk`: ^5.5.0 (terminal colors)
- `marked`: ^15.0.12 (markdown rendering)
- `mime-types`: ^3.0.1
- `get-east-asian-width`: ^1.3.0
- `koffi`: ^2.9.0 (optional native binding for VT input)
- Dev: `@xterm/xterm`: ^5.5.0, `@xterm/headless`: ^5.5.0

### 6. @mariozechner/pi-web-ui (0.62.0)
**Purpose:** Reusable web UI components for AI chat interfaces

Technology: Lit web components

Key dependencies:
- `@mariozechner/pi-ai`: ^0.62.0
- `@mariozechner/pi-tui`: ^0.62.0
- `lit`: ^3.3.1 (peer dependency)
- `lucide`: ^0.544.0 (icons)
- `pdfjs-dist`: 5.4.394 (PDF rendering)
- `docx-preview`: ^0.3.7 (Word document preview)
- `xlsx`: 0.20.3 (Excel/spreadsheet support)
- `@lmstudio/sdk`: ^1.5.0
- `ollama`: ^0.6.0

Build: TypeScript + TailwindCSS minification

### 7. @mariozechner/pi-pods (0.62.0)
**Purpose:** CLI for managing vLLM deployments on GPU pods

Dependencies:
- `@mariozechner/pi-agent-core`: ^0.62.0
- `chalk`: ^5.5.0

CLI: `pi-pods` (dist/cli.js)

---

## Build System

### Build Commands

| Command | Purpose |
|---------|---------|
| `npm run clean` | Remove all dist folders |
| `npm run build` | Sequential build of tui, ai, agent, coding-agent, mom, web-ui, pods |
| `npm run dev` | Concurrent dev mode for all packages |
| `npm run check` | Biome lint/format + TypeScript check + browser smoke test |
| `npm run test` | Run tests across all workspaces |

### Build Tool: tsgo

`tsgo` is a wrapper around `tsx` for TypeScript compilation. Each package builds via:
```bash
tsgo -p tsconfig.build.json
```

### Binary Compilation

Uses **Bun** to compile the coding-agent into standalone binaries:

```bash
bun build --compile ./dist/bun/cli.js --outfile dist/pi
```

Platforms: darwin-arm64, darwin-x64, linux-x64, linux-arm64, windows-x64

---

## Testing

| Package | Test Runner |
|---------|-------------|
| tui | Node.js built-in test (`node --test`) |
| ai | vitest |
| agent | vitest |
| coding-agent | vitest |

CI runs: `npm test`

---

## CI/CD

### GitHub Actions Workflows

| Workflow | Trigger | Purpose |
|----------|---------|---------|
| `ci.yml` | push to main, PRs to main | Build, check, test on Node 22 |
| `build-binaries.yml` | git tags v*, manual dispatch | Build cross-platform binaries with Bun |
| `pr-gate.yml` | PR opened | Check if contributor is approved, auto-close if not |
| `approve-contributor.yml` | Manual | Add approved contributors |
| `openclaw-gate.yml` | Manual | OpenClaw integration |
| `oss-weekend-issues.yml` | Scheduled | OSS weekend issue management |

### CI Environment

- **Runner:** ubuntu-latest
- **Node.js:** 22 (via actions/setup-node@v4)
- **System dependencies:** libcairo2-dev, libpango1.0-dev, libjpeg-dev, libgif-dev, librsvg2-dev, fd-find, ripgrep

---

## Version Management

- **Approach:** Shared version across all packages (monorepo versioning)
- **Script:** `sync-versions.js` keeps all package versions in sync
- **Release scripts:** `release:patch`, `release:minor`, `release:major`

---

## Dependency Overrides

```json
"overrides": {
  "rimraf": "6.1.2",
  "fast-xml-parser": "5.3.8",
  "gaxios": {
    "rimraf": "6.1.2"
  }
}
```

---

## Key Architectural Patterns

1. **Layered Architecture:**
   - `pi-ai` (LLM abstraction) -> `pi-agent-core` (agent runtime) -> `pi-coding-agent` (CLI)
   - `pi-tui` provides terminal rendering shared by multiple packages

2. **Transport Abstraction:** Agent core uses a transport abstraction for communication

3. **Tool-based Agent:** Coding agent implements read, bash, edit, write tools

4. **Provider Agnostic:** AI package unifies multiple LLM providers behind a common API

---

## Noteworthy Dependencies

| Dependency | Purpose |
|------------|---------|
| `@mariozechner/jiti` | TypeScript runtime compilation (2.6.5) |
| `@mariozechner/pi-coding-agent` | Published package referenced in root deps |
| `koffi` | Optional native binding (VT input on Windows) |
| `@silvia-odwyer/photon-node` | Rust-based image processing |
| `@anthropic-ai/sandbox-runtime` | Sandbox execution for mom bot |
| `partial-json` | Handle streaming JSON responses |
