# OpenClaw Tech Stack Analysis

## Project Overview

**OpenClaw** is a multi-channel AI gateway with extensible messaging integrations. The project is a sophisticated monorepo supporting:
- AI model routing and provisioning
- 50+ messaging channel integrations (Discord, Slack, Telegram, Matrix, WhatsApp, iMessage, etc.)
- Desktop/mobile applications (macOS, iOS, Android)
- Sandboxed agent execution
- Plugin SDK for third-party extensions

---

## 1. Monorepo Structure

### Package Manager: pnpm

**Version**: pnpm 10.32.1 (declared via `packageManager` field)

**Workspace Configuration** (`pnpm-workspace.yaml`):
```yaml
packages:
  - .           # Root (CLI, gateway, core)
  - ui          # Control UI (web interface)
  - packages/*  # Internal packages
  - extensions/* # Plugin extensions
```

### Build Dependencies (Native Modules)

**Explicitly built** (via `onlyBuiltDependencies`):
- `@lydell/node-pty` - PTY bindings
- `@matrix-org/matrix-sdk-crypto-nodejs` - Matrix encryption
- `@napi-rs/canvas` - Canvas rendering (used for A2UI)
- `@tloncorp/api` - Tlon platform API
- `@whiskeysockets/baileys` - WhatsApp Web protocol
- `authenticate-pam` - PAM authentication
- `esbuild` - Bundler
- `node-llama-cpp` - Local LLM inference
- `protobufjs` - Protocol buffers
- `sharp` - Image processing

**Ignored** (optional native deps):
- `@discordjs/opus` - Discord voice encoding
- `koffi` - FFI library

---

## 2. Runtime Environment

### Node.js

**Minimum**: Node.js 22.14.0
**Docker Base**: Node.js 24 (bookworm)

```bash
engines:
  node: ">=22.14.0"
```

### Bun Support

The project supports Bun as an alternative runtime:
- Build scripts use `bun` preferentially
- `OPENCLAW_PREFER_PNPM=1` env var can force pnpm for UI builds
- CI validates both Node 22 and Node 24 compatibility

### Platform Support

| Platform | Status | Notes |
|----------|--------|-------|
| Linux | Primary | Ubuntu 24.04 CI, Debian bookworm base |
| macOS | Supported | Intel + Apple Silicon |
| Windows | Tested | 32 vCPU runner in CI |
| Android | Supported | Gradle, Kotlin, Java 17 |
| iOS | Supported | Swift, Xcode 26.1 |

---

## 3. Language & Type System

### TypeScript

**Version**: TypeScript 5.9.3

**Configuration** (`tsconfig.json`):
```json
{
  "compilerOptions": {
    "target": "ES2023",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "strict": true,
    "noEmit": true,
    "noEmitOnError": true,
    "lib": ["DOM", "DOM.Iterable", "ES2023", "ScriptHost"],
    "paths": {
      "openclaw/plugin-sdk": ["./src/plugin-sdk/index.ts"],
      "openclaw/plugin-sdk/*": ["./src/plugin-sdk/*.ts"]
    }
  }
}
```

### Module System

- **ESM-first**: `"type": "module"` in package.json
- Path aliases via TypeScript for plugin SDK (`openclaw/plugin-sdk/*`)

---

## 4. Build System

### tsdown

**Version**: tsdown 0.21.4

**Primary bundler** for TypeScript compilation. Configuration in `tsdown.config.ts`:

**Build Entry Points**:
```typescript
// Core entries
index: "src/index.ts",
entry: "src/entry.ts",
extensionAPI: "src/extensionAPI.ts",

// Plugin SDK (60+ subpaths)
"plugin-sdk/*": "src/plugin-sdk/*.ts",

// Bundled plugins
"extensions/*/src/index.ts" (via listBundledPluginBuildEntries())

// CLI
"cli/daemon-cli": "src/cli/daemon-cli.ts",
"cli/memory-cli": "src/cli/memory-cli.ts",

// Bundled hooks
"bundled/*/handler": "src/hooks/bundled/*/handler.ts"
```

**Key Config**:
```typescript
platform: "node",
env: { NODE_ENV: "production" },
deps: { neverBundle: ["@lancedb/lancedb"] }
```

### Build Scripts

```bash
pnpm build           # Full build (tsdown + UI + postbuild)
pnpm build:docker    # Docker-optimized build (no UI bundler)
pnpm build:strict-smoke  # Type check + DTS generation
```

### Post-build Scripts

1. `scripts/tsdown-build.mjs` - Run tsdown
2. `scripts/runtime-postbuild.mjs` - Runtime-specific fixes
3. `scripts/build-stamp.mjs` - Build metadata injection
4. `pnpm build:plugin-sdk:dts` - Generate TypeScript declarations
5. `scripts/canvas-a2ui-copy.ts` - Copy A2UI bundle
6. `scripts/copy-hook-metadata.ts` - Hook metadata
7. `scripts/copy-export-html-templates.ts` - HTML templates
8. `scripts/write-plugin-sdk-entry-dts.ts` - Plugin SDK entry DTS
9. `scripts/write-build-info.ts` - Build info
10. `scripts/write-cli-startup-metadata.ts` - CLI startup metadata
11. `scripts/write-cli-compat.ts` - CLI compatibility layer

---

## 5. Testing Stack

### Vitest

**Version**: Vitest 4.1.0
**Coverage Provider**: V8

### Test Configurations

| Config File | Purpose |
|-------------|---------|
| `vitest.config.ts` | Main unit tests |
| `vitest.unit.config.ts` | Unit test subset |
| `vitest.gateway.config.ts` | Gateway-specific tests |
| `vitest.e2e.config.ts` | End-to-end tests |
| `vitest.live.config.ts` | Live/integration tests |
| `vitest.channels.config.ts` | Channel tests |
| `vitest.extensions.config.ts` | Extension tests |
| `vitest.performance-config.ts` | Performance benchmarks |
| `ui/vitest.config.ts` | UI component tests |
| `ui/vitest.node.config.ts` | UI Node tests |

### Test Configuration Highlights

```typescript
// From vitest.config.ts
pool: "forks",           // NOT threads/vmForks
maxWorkers: isCI ? 3 : localWorkers,  // Adaptive based on host
testTimeout: 120_000,
hookTimeout: isWindows ? 180_000 : 120_000,
unstubEnvs: true,        // Automatic env restoration
unstubGlobals: true,     // Avoid cross-test pollution
```

### Coverage Thresholds

```yaml
lines: 70%
functions: 70%
branches: 55%
statements: 70%
```

### Key Testing Patterns

- **Test files**: Colocated `*.test.ts` next to source
- **E2E files**: `*.e2e.test.ts` naming convention
- **Live tests**: `*.live.test.ts` (require `OPENCLAW_LIVE_TEST=1`)
- **Memory hotspots**: Manifest-based testing at `scripts/test-runner-manifest.mjs`

---

## 6. Linting & Formatting

### Oxlint + Oxfmt

**Versions**:
- oxlint: 1.56.0
- oxfmt: 0.41.0
- oxlint-tsgolint: 0.17.1

### Pre-commit Hooks (.pre-commit-config.yaml)

**Hooks**:
| Hook | Tool | Purpose |
|------|------|---------|
| File hygiene | pre-commit-hooks | trailing-whitespace, end-of-file-fixer, etc. |
| Secret detection | detect-secrets | Yelp/detect-secrets |
| Shell linting | shellcheck | Shell script validation |
| GitHub Actions | actionlint | Workflow validation |
| Actions security | zizmor | Security audit for workflows |
| Python linting | ruff | Python skills linting |
| Python tests | pytest | Python skills tests |
| pnpm audit | pnpm audit | Production dependency audit |
| TypeScript lint | oxlint | JS/TS linting |
| TypeScript fmt | oxfmt | Formatting |
| Swift lint | swiftlint | iOS/macOS linting |
| Swift fmt | swiftformat | iOS/macOS formatting |

---

## 7. Web Framework

### Express 5.x

**Version**: Express 5.2.1

Primary HTTP framework for the gateway server and API routes.

### Hono 4.x

**Version**: Hono 4.12.8

Used for lightweight API routes and middleware. Fast, edge-compatible framework.

### Other HTTP-related Dependencies

- `ws` 8.20.0 - WebSocket server
- `undici` 7.24.5 - HTTP client
- `@sinclair/typebox` 0.34.48 - JSON schema type builder
- `ajv` 8.18.0 - JSON schema validation

---

## 8. Database & Storage

### SQLite Vec

**Version**: sqlite-vec 0.1.7

Vector similarity search via SQLite extension.

### LanceDB

**Referenced** in tsdown config (`neverBundle`):
```typescript
deps: { neverBundle: ["@lancedb/lancedb"] }
```

Memory/embedding storage for RAG workflows.

### JSON Store

Custom JSON file-based storage (`openclaw/plugin-sdk/json-store`).

---

## 9. Messaging & Chat Integrations

### Channel SDKs

| Platform | SDK/Protocol |
|----------|--------------|
| Discord | `discord.js` ecosystem |
| Slack | `@slack/web-api` + Bolt |
| Telegram | `grammy` ( grammY framework) |
| Matrix | `@matrix-org/matrix-sdk` |
| WhatsApp | `@whiskeysockets/baileys` |
| iMessage | BlueBubbles protocol |
| LINE | `@line/bot-sdk` 10.6.0 |
| Microsoft Teams | REST API |
| IRC | irc-framework |
| Signal | signal-whisper |
| Feishu | REST API |
| Google Chat | REST API |
| Nextcloud Talk | REST API |
| Nostr | nostr-tools |
| Zalo | REST API |
| Mattermost | REST API |
| Twitch | tmi.js |
| Tlon | `@tloncorp/api` |

### Messaging Protocol

- **Webhooks**: Inbound webhook handling (`openclaw/plugin-sdk/webhook-ingress`)
- **WebSocket**: Real-time messaging via `ws`
- **Long Polling**: Fallback for Matrix

---

## 10. AI/ML Integrations

### Provider SDKs

| Provider | Package |
|----------|---------|
| OpenAI | `openai` (official) |
| Anthropic | `@anthropic-ai/sdk` + Vertex |
| Google | `@google/generative-ai` + Vertex |
| Azure OpenAI | Azure SDK |
| AWS Bedrock | `@aws-sdk/client-bedrock` |
| Ollama | REST API (local) |
| LM Studio | REST API (local) |
| vLLM | REST API (local) |
| Local GGUF | `node-llama-cpp` |

### AI Framework

- **MCP SDK**: `@modelcontextprotocol/sdk` 1.27.1
- **Pi Agent**: `@mariozechner/pi-agent-core` 0.61.1
- **Agent Client Protocol**: `@agentclientprotocol/sdk` 0.16.1

### Media Understanding

- `pdfjs-dist` 5.5.207 - PDF parsing
- `@mozilla/readability` 0.6.0 - Article extraction
- `linkedom` 0.18.12 - DOM parsing
- `sharp` 0.34.5 - Image processing
- `playwright-core` 1.58.2 - Browser automation

### Speech/TTS

- `node-edge-tts` 1.2.10 - Microsoft Edge TTS
- `elevenlabs` SDK - Voice synthesis

---

## 11. Docker & Containerization

### Base Images

| Image | Purpose |
|-------|---------|
| `node:24-bookworm` | Default runtime |
| `node:24-bookworm-slim` | Slim variant |
| `debian:bookworm-slim` | Sandbox base |

### Multi-stage Dockerfile

**Stages**:
1. `ext-deps` - Extract extension package.json files
2. `build` - Install dependencies, run build
3. `runtime-assets` - Prune dev deps, strip type files
4. `base-default` / `base-slim` - Runtime base
5. `runtime` - Final runtime image

### Build Args

```dockerfile
OPENCLAW_EXTENSIONS=""      # Opt-in extensions
OPENCLAW_VARIANT="default"  # or "slim"
OPENCLAW_INSTALL_BROWSER="" # Install Chromium + Xvfb
OPENCLAW_INSTALL_DOCKER_CLI="" # Docker CLI for sandbox
OPENCLAW_DOCKER_APT_PACKAGES="" # Extra system packages
```

### Docker Compose

**Services**:
- `openclaw-gateway` - Gateway server (port 18789)
- `openclaw-cli` - CLI client (networked to gateway)

---

## 12. Mobile Development

### Android

**Build System**: Gradle 8.11.1
**Language**: Kotlin
**Min SDK**: Configured in `apps/android/app/build.gradle.kts`
**Java Version**: JDK 17

**CI Tasks**:
```bash
android:assemble       # Debug APK
android:bundle:release # AAB release
android:test           # Unit tests
android:test:integration # Live integration tests
```

### iOS/macOS

**Build System**: Swift Package Manager + XcodeGen
**Language**: Swift
**Xcode**: 26.1

**CI Tasks**:
```bash
ios:build              # iOS simulator build
mac:package           # macOS app bundle
swiftlint             # Linting
swiftformat --lint    # Formatting
swift build           # Package build
swift test            # Tests
```

---

## 13. Python Integration

### Skills Scripts

Python-based agent skills in `skills/` directory.

**Tooling**:
- `ruff` 0.14.1 - Linting (via pre-commit)
- `pytest` - Testing
- Python 3.12 - Runtime

**Configuration** (`pyproject.toml`):
```toml
[tool.ruff]
target-version = "py310"
line-length = 100

[tool.pytest.ini_options]
testpaths = ["skills"]
python_files = ["test_*.py"]
```

---

## 14. Skills & Agents

### Agent Skills (.agents/skills/)

Located in `.agents/skills/`:

| Skill | Purpose |
|-------|---------|
| `openclaw-release-maintainer` | Release naming, version coordination |
| `openclaw-ghsa-maintainer` | GHSA advisory workflow |
| `openclaw-pr-maintainer` | PR triage, review, landing |
| `openclaw-parallels-smoke` | Cross-platform smoke tests |
| `parallels-discord-roundtrip` | Discord roundtrip testing |
| `openclaw-test-heap-leaks` | Memory leak detection |
| `security-triage` | Security issue triage |

### Runtime Skills

Skills installed via `skills/` directory with Python scripts and bundled capabilities.

---

## 15. Key Libraries Summary

### Core Runtime

| Library | Version | Purpose |
|---------|---------|---------|
| express | 5.2.1 | HTTP server |
| hono | 4.12.8 | Lightweight API |
| ws | 8.20.0 | WebSocket |
| yaml | 2.8.3 | Config parsing |
| zod | 4.3.6 | Schema validation |
| dotenv | 17.3.1 | Env vars |
| jiti | 2.6.1 | TypeScript execution |
| tslog | 4.10.2 | Logging |
| undici | 7.24.5 | HTTP client |

### Data Processing

| Library | Version | Purpose |
|---------|---------|---------|
| json5 | 2.2.3 | JSON parsing |
| jszip | 3.10.1 | ZIP handling |
| tar | 7.5.12 | Archive handling |
| file-type | 21.3.4 | File type detection |
| long | 5.3.2 | 64-bit integers |
| uuid | 13.0.0 | UUID generation |

### CLI & UI

| Library | Version | Purpose |
|---------|---------|---------|
| commander | 14.0.3 | CLI framework |
| chalk | 5.6.2 | Terminal colors |
| cli-highlight | 2.1.11 | Syntax highlighting |
| qrcode-terminal | 0.12.0 | QR code display |
| @clack/prompts | 1.1.0 | Interactive prompts |
| osc-progress | 0.3.0 | Progress bars |

### Testing & Dev

| Library | Version | Purpose |
|---------|---------|---------|
| vitest | 4.1.0 | Test runner |
| @vitest/coverage-v8 | 4.1.0 | V8 coverage |
| jsdom | 29.0.1 | DOM simulation |
| jscpd | 4.0.8 | Copy-paste detection |
| tsx | 4.21.0 | TypeScript execution |
| typescript | 5.9.3 | Type checker |

---

## 16. CI/CD Infrastructure

### GitHub Actions Workflows

| Workflow | Purpose |
|----------|---------|
| `ci.yml` | Main CI pipeline |
| `ci-bun.yml` | Bun-specific tests |
| `codeql.yml` | CodeQL security analysis |
| `docker-release.yml` | Docker image publishing |
| `macos-release.yml` | macOS app release |
| `openclaw-npm-release.yml` | NPM package release |
| `plugin-npm-release.yml` | Plugin NPM releases |
| `stale.yml` | Stale issue management |
| `labeler.yml` | PR label automation |
| `auto-response.yml` | Automated responses |

### CI Architecture

**Preflight System**: Dynamic matrix generation based on changed files:
- Docs-only detection
- Scope detection (Node, macOS, Android, Windows)
- Extension change detection
- Matrix output via `scripts/ci-write-manifest-outputs.mjs`

**Runners**:
- `blacksmith-16vcpu-ubuntu-2404` - Primary Linux
- `blacksmith-32vcpu-windows-2025` - Windows
- `macos-latest` - macOS

---

## 17. Path Aliases

```typescript
// tsconfig.json paths
"openclaw/extension-api": ["./src/extensionAPI.ts"]
"openclaw/plugin-sdk": ["./src/plugin-sdk/index.ts"]
"openclaw/plugin-sdk/*": ["./src/plugin-sdk/*.ts"]
"openclaw/plugin-sdk/account-id": ["./src/plugin-sdk/account-id.ts"]
```

---

## 18. Key Architectural Decisions

1. **ESM-only**: No CommonJS, strict ESM throughout
2. **TypeScript strict mode**: No `any`, no `@ts-nocheck`
3. **Vitest forks pool only**: Avoids `threads`/`vmForks` instability
4. **pnpm workspaces**: Monorepo with shared dependencies
5. **Plugin SDK boundary**: Extensions must use `openclaw/plugin-sdk/*` imports
6. **Native module handling**: Explicit build tracking for problematic deps
7. **A2UI bundle**: Pre-bundled Angular 2 UI components
8. **Multi-platform CI**: Simultaneous testing on Linux, macOS, Windows, Android
9. **Corepack enabled**: Ensures consistent pnpm version
10. **No dynamic import mixing**: Guardrail against lazy loading issues

---

## Summary Table

| Category | Technology |
|----------|------------|
| Language | TypeScript 5.9 (ESM) |
| Runtime | Node 22+ / Bun |
| Package Manager | pnpm 10.32 |
| Bundler | tsdown + esbuild |
| Test Runner | Vitest 4.1 (V8 coverage) |
| Linter | Oxlint 1.56 |
| Formatter | Oxfmt 0.41 |
| Web Framework | Express 5 + Hono 4 |
| Container | Docker (Node 24 bookworm) |
| Mobile | Android (Gradle/Kotlin), iOS (Swift/Xcode) |
| Python | 3.12 + ruff + pytest |
| Mobile CI | Xcode 26.1, Java 17 |
