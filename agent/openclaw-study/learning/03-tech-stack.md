# OpenClaw Tech Stack

**Study Note:** Synthesized from research/02-tech-stack.md
**Project:** OpenClaw - Personal AI Assistant
**Date:** 2026-03-26

---

## 1. Language and Runtime

### TypeScript

**Version:** TypeScript 5.9.3

**Configuration:**
```json
{
  "compilerOptions": {
    "target": "ES2023",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "strict": true,
    "noEmit": true,
    "noEmitOnError": true,
    "lib": ["DOM", "DOM.Iterable", "ES2023", "ScriptHost"]
  }
}
```

**Key Decisions:**
- ESM-only (`"type": "module"` in package.json)
- Strict mode enforced (no `any`, no `@ts-nocheck`)
- Path aliases: `openclaw/plugin-sdk/*` maps to `src/plugin-sdk/*.ts`

### Node.js

**Minimum:** Node.js 22.14.0
**Docker Base:** Node.js 24 (bookworm)

**Platform Support:**
| Platform | Status | Notes |
|----------|--------|-------|
| Linux | Primary | Ubuntu 24.04 CI, Debian bookworm |
| macOS | Supported | Intel + Apple Silicon |
| Windows | Tested | 32 vCPU runner in CI |
| Android | Supported | Gradle, Kotlin, Java 17 |
| iOS | Supported | Swift, Xcode 26.1 |

**Bun Support:** Project supports Bun as alternative runtime with `OPENCLAW_PREFER_PNPM=1` env var.

---

## 2. Build System

### pnpm Workspaces

**Version:** pnpm 10.32.1

**Workspace Configuration:**
```yaml
packages:
  - .           # Root (CLI, gateway, core)
  - ui          # Control UI (web interface)
  - packages/*  # Internal packages
  - extensions/* # Plugin extensions
```

### tsdown

**Version:** tsdown 0.21.4

Primary bundler for TypeScript compilation. Configuration in `tsdown.config.ts`:

**Build Entry Points:**
```typescript
// Core entries
index: "src/index.ts",
entry: "src/entry.ts",
extensionAPI: "src/extensionAPI.ts",

// Plugin SDK (60+ subpaths)
"plugin-sdk/*": "src/plugin-sdk/*.ts",

// CLI
"cli/daemon-cli": "src/cli/daemon-cli.ts",
"cli/memory-cli": "src/cli/memory-cli.ts",

// Bundled hooks
"bundled/*/handler": "src/hooks/bundled/*/handler.ts"
```

**Key Config:**
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

### Native Module Handling

**Explicitly built** (via `onlyBuiltDependencies`):
- `@lydell/node-pty` - PTY bindings
- `@matrix-org/matrix-sdk-crypto-nodejs` - Matrix encryption
- `@napi-rs/canvas` - Canvas rendering
- `@whiskeysockets/baileys` - WhatsApp Web protocol
- `authenticate-pam` - PAM authentication
- `esbuild` - Bundler
- `node-llama-cpp` - Local LLM inference
- `protobufjs` - Protocol buffers
- `sharp` - Image processing

---

## 3. Web Framework

### Express 5.x

**Version:** Express 5.2.1

Primary HTTP framework for the gateway server and API routes.

### Hono 4.x

**Version:** Hono 4.12.8

Used for lightweight API routes and middleware. Fast, edge-compatible framework.

### WebSocket

**Library:** ws 8.20.0

Real-time bidirectional communication between Gateway and clients.

### HTTP Utilities

| Library | Version | Purpose |
|---------|---------|---------|
| undici | 7.24.5 | HTTP client |
| @sinclair/typebox | 0.34.48 | JSON schema type builder |
| ajv | 8.18.0 | JSON schema validation |

---

## 4. Testing Stack

### Vitest

**Version:** Vitest 4.1.0
**Coverage Provider:** V8

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

### Test Configuration Highlights

```typescript
pool: "forks",           // NOT threads/vmForks
maxWorkers: isCI ? 3 : localWorkers,  // Adaptive
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

### Test File Conventions

- Unit tests: Colocated `*.test.ts` next to source
- E2E tests: `*.e2e.test.ts` naming convention
- Live tests: `*.live.test.ts` (require `OPENCLAW_LIVE_TEST=1`)

---

## 5. Linting

### Oxlint + Oxfmt

**Versions:**
- oxlint: 1.56.0
- oxfmt: 0.41.0
- oxlint-tsgolint: 0.17.1

### Pre-commit Hooks

| Hook | Tool | Purpose |
|------|------|---------|
| File hygiene | pre-commit-hooks | trailing-whitespace, end-of-file-fixer |
| Secret detection | detect-secrets | Yelp/detect-secrets |
| Shell linting | shellcheck | Shell script validation |
| GitHub Actions | actionlint | Workflow validation |
| Actions security | zizmor | Security audit for workflows |
| Python linting | ruff | Python skills linting |
| TypeScript lint | oxlint | JS/TS linting |
| TypeScript fmt | oxfmt | Formatting |
| Swift lint | swiftlint | iOS/macOS linting |
| Swift fmt | swiftformat | iOS/macOS formatting |

---

## 6. Containerization

### Docker Multi-stage Build

**Base Images:**
| Image | Purpose |
|-------|---------|
| `node:24-bookworm` | Default runtime |
| `node:24-bookworm-slim` | Slim variant |
| `debian:bookworm-slim` | Sandbox base |

**Build Stages:**
1. `ext-deps` - Extract extension package.json files
2. `build` - Install dependencies, run build
3. `runtime-assets` - Prune dev deps, strip type files
4. `base-default` / `base-slim` - Runtime base
5. `runtime` - Final runtime image

**Build Arguments:**
```dockerfile
OPENCLAW_EXTENSIONS=""      # Opt-in extensions
OPENCLAW_VARIANT="default"  # or "slim"
OPENCLAW_INSTALL_BROWSER="" # Install Chromium + Xvfb
OPENCLAW_INSTALL_DOCKER_CLI="" # Docker CLI for sandbox
OPENCLAW_DOCKER_APT_PACKAGES="" # Extra system packages
```

---

## 7. Mobile Development

### Android

| Component | Technology |
|-----------|------------|
| Build System | Gradle 8.11.1 |
| Language | Kotlin |
| Min SDK | Configured in `apps/android/app/build.gradle.kts` |
| Java Version | JDK 17 |

**CI Tasks:**
```bash
android:assemble       # Debug APK
android:bundle:release # AAB release
android:test           # Unit tests
android:test:integration # Live integration tests
```

### iOS/macOS

| Component | Technology |
|-----------|------------|
| Build System | Swift Package Manager + XcodeGen |
| Language | Swift |
| Xcode | 26.1 |

**CI Tasks:**
```bash
ios:build              # iOS simulator build
mac:package           # macOS app bundle
swiftlint             # Linting
swiftformat --lint    # Formatting
swift build           # Package build
swift test            # Tests
```

---

## 8. Key Architectural Decisions

| Decision | Rationale |
|----------|-----------|
| ESM-only | No CommonJS, strict ESM throughout |
| TypeScript strict mode | No `any`, no `@ts-nocheck` |
| Vitest forks pool only | Avoids `threads`/`vmForks` instability |
| pnpm workspaces | Monorepo with shared dependencies |
| Plugin SDK boundary | Extensions must use `openclaw/plugin-sdk/*` imports only |
| Native module handling | Explicit build tracking for problematic deps |
| A2UI bundle | Pre-bundled Angular 2 UI components |
| Multi-platform CI | Simultaneous testing on Linux, macOS, Windows, Android |
| Corepack enabled | Ensures consistent pnpm version |
| No dynamic import mixing | Guardrail against lazy loading issues |

---

## 9. Core Libraries Summary

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

### CLI & UI

| Library | Version | Purpose |
|---------|---------|---------|
| commander | 14.0.3 | CLI framework |
| chalk | 5.6.2 | Terminal colors |
| cli-highlight | 2.1.11 | Syntax highlighting |
| qrcode-terminal | 0.12.0 | QR code display |
| @clack/prompts | 1.1.0 | Interactive prompts |
| osc-progress | 0.3.0 | Progress bars |

### AI/Agent

| Library | Version |
|---------|---------|
| @mariozechner/pi-agent-core | 0.61.1 |
| @mariozechner/pi-ai | 0.61.1 |
| @modelcontextprotocol/sdk | 1.27.1 |

### Data Processing

| Library | Version | Purpose |
|---------|---------|---------|
| json5 | 2.2.3 | JSON parsing |
| jszip | 3.10.1 | ZIP handling |
| tar | 7.5.12 | Archive handling |
| file-type | 21.3.4 | File type detection |
| uuid | 13.0.0 | UUID generation |

---

## 10. Tech Stack Summary Table

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
