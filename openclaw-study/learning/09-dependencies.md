# OpenClaw Dependencies and Architecture

## Monorepo Structure

**Package Manager**: pnpm 10.32.1 (declared via `packageManager` field)

**Workspace Configuration** (`pnpm-workspace.yaml`):
```yaml
packages:
  - .           # Root (CLI, gateway, core)
  - ui          # Control UI (web interface)
  - packages/*  # Internal packages
  - extensions/* # Plugin extensions
```

## Build Dependencies (Native Modules)

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

## Runtime Environment

**Node.js**: Minimum 22.14.0, Docker base uses Node 24 (bookworm)

**Bun Support**: The project supports Bun as an alternative runtime with `OPENCLAW_PREFER_PNPM=1` to force pnpm for UI builds.

**Platform Support**:
| Platform | Status | Notes |
|----------|--------|-------|
| Linux | Primary | Ubuntu 24.04 CI, Debian bookworm base |
| macOS | Supported | Intel + Apple Silicon |
| Windows | Tested | 32 vCPU runner in CI |
| Android | Supported | Gradle, Kotlin, Java 17 |
| iOS | Supported | Swift, Xcode 26.1 |

## TypeScript Configuration

**Version**: TypeScript 5.9.3

**Key settings**:
- `target: ES2023`, `module: NodeNext`, `moduleResolution: NodeNext`
- `strict: true`, `noEmit: true`, `noEmitOnError: true`
- Path aliases: `openclaw/plugin-sdk/*` maps to `./src/plugin-sdk/*.ts`

**Module System**: ESM-first with `"type": "module"` in package.json

## Build System

**tsdown** (v0.21.4) is the primary bundler for TypeScript compilation.

**Build entry points**:
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

**Post-build scripts** (11 steps):
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

## Testing Stack

**Vitest** (v4.1.0) with V8 coverage provider.

**Test configurations**:
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

**Key settings**: `pool: "forks"`, adaptive workers, `testTimeout: 120_000`, automatic env/globals restoration

**Coverage thresholds**: 70% lines/functions/statements, 55% branches

## Web Framework

| Library | Version | Purpose |
|---------|--------|---------|
| express | 5.2.1 | HTTP server (primary) |
| hono | 4.12.8 | Lightweight API routes |
| ws | 8.20.0 | WebSocket server |
| undici | 7.24.5 | HTTP client |
| `@sinclair/typebox` | 0.34.48 | JSON schema type builder |
| ajv | 8.18.0 | JSON schema validation |

## Database & Storage

- **sqlite-vec** (v0.1.7) - Vector similarity search via SQLite extension
- **LanceDB** - Memory/embedding storage for RAG workflows (never bundled)
- **JSON Store** - Custom JSON file-based storage

## Key Dependencies Summary

**Core Runtime**:
| Library | Version | Purpose |
|---------|--------|---------|
| express | 5.2.1 | HTTP server |
| hono | 4.12.8 | Lightweight API |
| ws | 8.20.0 | WebSocket |
| yaml | 2.8.3 | Config parsing |
| zod | 4.3.6 | Schema validation |
| dotenv | 17.3.1 | Env vars |
| jiti | 2.6.1 | TypeScript execution |
| tslog | 4.10.2 | Logging |
| undici | 7.24.5 | HTTP client |

**CLI & UI**:
| Library | Version | Purpose |
|---------|--------|---------|
| commander | 14.0.3 | CLI framework |
| chalk | 5.6.2 | Terminal colors |
| @clack/prompts | 1.1.0 | Interactive prompts |
| osc-progress | 0.3.0 | Progress bars |

**Data Processing**:
| Library | Version | Purpose |
|---------|--------|---------|
| json5 | 2.2.3 | JSON parsing |
| jszip | 3.10.1 | ZIP handling |
| tar | 7.5.12 | Archive handling |
| file-type | 21.3.4 | File type detection |
| uuid | 13.0.0 | UUID generation |

## Interesting Dependency Choices

1. **tsdown over tsc/tsup**: Uses tsdown for bundling instead of more common choices like tsup or rollup
2. **Oxlint + Oxfmt**: Custom linter/formatter from the same maintainer as tsdown
3. **node:sqlite over better-sqlite3**: Uses Node.js built-in sqlite module with sqlite-vec extension for vector search
4. **pnpm over npm/yarn**: pnpm with strict workspace isolation
5. **jiti for TS execution**: Uses jiti for runtime TypeScript execution in CLI contexts
6. **Vitest forks pool**: Uses `forks` pool explicitly (not `threads` or `vmForks`) to avoid instability
7. **TypeBox for schema**: Uses `@sinclair/typebox` for JSON schema types instead of Zod schemas directly

## CI/CD Infrastructure

**Workflows**:
- `ci.yml` - Main CI pipeline
- `ci-bun.yml` - Bun-specific tests
- `codeql.yml` - CodeQL security analysis
- `docker-release.yml` - Docker image publishing
- `macos-release.yml` - macOS app release
- `openclaw-npm-release.yml` - NPM package release
- `plugin-npm-release.yml` - Plugin NPM releases

**Preflight System**: Dynamic matrix generation based on changed files with scope detection.

## Key Architectural Decisions

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
