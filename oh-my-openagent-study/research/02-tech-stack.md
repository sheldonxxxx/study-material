# Tech Stack Analysis: oh-my-openagent

## Overview

| Property | Value |
|----------|-------|
| **Project Name** | oh-my-opencode / oh-my-openagent |
| **Version** | 3.11.0 |
| **Type** | TypeScript monorepo with platform binaries |
| **Primary Runtime** | Bun 1.x |
| **Language** | TypeScript (strict mode, ESNext) |

---

## 1. Runtime & Package Management

### Runtime
- **Bun** — Used as the primary runtime, build tool, and package manager
  - All CI workflows use `oven-sh/setup-bun@v2` with `bun-version: latest`
  - Build commands use `bun build` for compilation
  - Tests run via `bun test`
  - Type checking via `bun run typecheck`

### Package Manager
- **Bun** (native workspace support)
  - Lock file: `bun.lock` (lockfileVersion 1)
  - Simple workspace configuration (not Turborepo, Nx, or Lerna)
  - `trustedDependencies` declared for native modules: `@ast-grep/cli`, `@ast-grep/napi`, `@code-yeongyu/comment-checker`

---

## 2. Language & TypeScript Configuration

### TypeScript
- **Version**: `^5.7.3`
- **Target**: `ESNext`
- **Module**: `ESNext`
- **Module Resolution**: `bundler`
- **Strict Mode**: Enabled
- **Declaration**: Enabled with separate emit via `tsc --emitDeclarationOnly`

```json
// tsconfig.json
{
  "compilerOptions": {
    "target": "ESNext",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "declaration": true,
    "declarationDir": "dist",
    "skipLibCheck": true,
    "esModuleInterop": true,
    "lib": ["ESNext"],
    "types": ["bun-types"]
  }
}
```

### Type Definitions
- `bun-types: 1.3.10` — Provides Bun-specific type definitions

---

## 3. Build System

### Build Commands

| Command | Purpose |
|---------|---------|
| `bun run build` | Compiles src/index.ts and src/cli/index.ts via bun, emits declarations via tsc |
| `bun run build:binaries` | Builds platform-specific compiled binaries via `script/build-binaries.ts` |
| `bun run build:schema` | Generates JSON schema for plugin configuration |
| `bun run build:model-capabilities` | Generates model capability definitions |
| `bun run build:all` | Runs build + build:binaries |

### Build Output
- **Main package**: `dist/index.js` (ESM), `dist/index.d.ts` (declarations)
- **CLI**: `dist/cli/index.js`
- **Bin entry**: `bin/oh-my-opencode.js`

### Binary Compilation
- Uses `bun build --compile --minify --target=<platform>` to produce standalone executables
- 11 platform targets across macOS (arm64, x64), Linux (glibc, musl), and Windows

---

## 4. Monorepo Structure

### Workspace Model
- **Type**: Simple Bun workspace (no Turborepo/Nx/Lerna)
- **Root**: `oh-my-opencode` (main npm package)
- **Platform packages**: 11 separate npm packages under `packages/`

### Platform Packages

| Package | OS | CPU | Libc |
|---------|----|----|-----|
| `oh-my-opencode-darwin-arm64` | darwin | arm64 | - |
| `oh-my-opencode-darwin-x64` | darwin | x64 | - |
| `oh-my-opencode-darwin-x64-baseline` | darwin | x64 | - |
| `oh-my-opencode-linux-x64` | linux | x64 | glibc |
| `oh-my-opencode-linux-x64-baseline` | linux | x64 | glibc |
| `oh-my-opencode-linux-arm64` | linux | arm64 | glibc |
| `oh-my-opencode-linux-x64-musl` | linux | x64 | musl |
| `oh-my-opencode-linux-x64-musl-baseline` | linux | x64 | musl |
| `oh-my-opencode-linux-arm64-musl` | linux | arm64 | musl |
| `oh-my-opencode-windows-x64` | windows | x64 | - |
| `oh-my-opencode-windows-x64-baseline` | windows | x64 | - |

### Package Publishing
- Main package and platform binaries are published separately to npm
- Two package names: `oh-my-opencode` and `oh-my-openagent` (aliased)
- Both published with same version from same codebase

---

## 5. Key Dependencies

### Production Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| `@ast-grep/napi` | `^0.41.1` | AST analysis and transformation |
| `@ast-grep/cli` | `^0.41.1` | CLI for ast-grep |
| `@opencode-ai/sdk` | `^1.2.24` | OpenCode SDK |
| `@opencode-ai/plugin` | `^1.2.24` | OpenCode plugin interface |
| `@modelcontextprotocol/sdk` | `^1.25.2` | MCP (Model Context Protocol) |
| `zod` | `^4.1.8` | Schema validation |
| `commander` | `^14.0.2` | CLI argument parsing |
| `@clack/prompts` | `^0.11.0` | Terminal prompts UI |
| `picocolors` | `^1.1.1` | Terminal colors |
| `picomatch` | `^4.0.2` | Glob pattern matching |
| `diff` | `^8.0.3` | Text diffing |
| `js-yaml` | `^4.1.1` | YAML parsing |
| `jsonc-parser` | `^3.3.1` | JSONC parsing |
| `detect-libc` | `^2.0.0` | Detect C library version |
| `vscode-jsonrpc` | `^8.2.0` | JSON-RPC implementation |

### Dev Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| `typescript` | `^5.7.3` | TypeScript compiler |
| `bun-types` | `1.3.10` | Bun type definitions |
| `@types/js-yaml` | `^4.0.9` | Type definitions |
| `@types/picomatch` | `^3.0.2` | Type definitions |

---

## 6. CI/CD Configuration

### Workflows (GitHub Actions)

| Workflow | Trigger | Purpose |
|----------|---------|---------|
| `ci.yml` | push (master, dev), PR | Run tests, typecheck, build |
| `publish.yml` | workflow_dispatch | Publish to npm (main + platform packages) |
| `publish-platform.yml` | workflow_call / dispatch | Build and publish platform binaries |
| `lint-workflows.yml` | push, PR | Lint GitHub workflow files |
| `cla.yml` | pull_request | CLA assistant |
| `sisyphus-agent.yml` | schedule (daily) | Daily Sisyphus agent task |
| `refresh-model-capabilities.yml` | schedule (weekly) | Refresh model capabilities |

### CI Jobs (ci.yml)
1. **block-master-pr** — Block PRs targeting master branch
2. **test** — Run bun test (split into mock-heavy and remaining tests)
3. **typecheck** — Run `tsc --noEmit`
4. **build** — Run `bun run build` and verify output
5. **draft-release** — Create/update draft release on dev branch

### Test Strategy
- Mock-heavy tests run in isolation (separate `bun test` processes) to prevent module cache pollution
- Tests preload: `./test-setup.ts` (configured in `bunfig.toml`)

---

## 7. Project Structure

```
oh-my-openagent/
├── src/                          # Main TypeScript source
│   ├── index.ts                  # Plugin entry point
│   ├── agents/                   # 11 agent implementations
│   ├── hooks/                    # 48 lifecycle hooks
│   ├── tools/                    # 26 tools
│   ├── features/                 # 19 feature modules
│   ├── shared/                   # 95+ utilities
│   ├── config/                   # Zod schema system
│   ├── cli/                      # CLI commands (Commander.js)
│   ├── mcp/                      # Built-in MCP servers
│   ├── plugin/                   # OpenCode hook handlers
│   └── plugin-handlers/          # 6-phase config pipeline
├── packages/                     # Platform binary packages
│   ├── darwin-arm64/
│   ├── darwin-x64/
│   ├── linux-x64/
│   ├── linux-arm64/
│   ├── windows-x64/
│   └── ... (11 total)
├── script/                       # Build and release scripts
├── bin/                          # CLI entry point
├── test-setup.ts                 # Test preload configuration
├── bunfig.toml                   # Bun configuration
├── tsconfig.json                 # TypeScript configuration
└── package.json                   # Root package definition
```

---

## 8. Configuration Files

| File | Purpose |
|------|---------|
| `bunfig.toml` | Bun test preload: `./test-setup.ts` |
| `tsconfig.json` | TypeScript compiler options |
| `package.json` | Package definition, scripts, dependencies |

---

## 9. Artifact Summary

| Artifact Type | Location |
|--------------|----------|
| Source | `src/` (1268 TypeScript files, ~160k LOC) |
| Distribution | `dist/` (JS + declarations) |
| Platform Binaries | `packages/*/bin/` |
| Lock File | `bun.lock` |
| Schema | `assets/oh-my-opencode.schema.json` |

---

## 10. Notable Technical Choices

1. **Bun as unified toolchain** — Replaces npm/yarn, tsc, jest, ts-node with single runtime
2. **No monorepo tooling** — Uses native Bun workspaces instead of Turborepo/Nx/Lerna
3. **Platform-specific binaries** — 11 pre-compiled binaries for cross-platform distribution
4. **Baseline builds** — Separate "baseline" targets for older glibc/musl versions
5. **Dual npm packages** — `oh-my-opencode` and `oh-my-openagent` (alias) both published
6. **Mock isolation** — Tests using `mock.module()` run in separate processes
7. **Zod v4** — Using cutting-edge Zod with v4 schema syntax
8. **MCP integration** — First-class support for Model Context Protocol servers
