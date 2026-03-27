# Project Overview: oh-my-openagent

## Project Summary

**oh-my-openagent** (v3.11.0) is an AI Agent Harness with Multi-Model Orchestration, Parallel Background Agents, and LSP/AST Tools. It is a monorepo Node.js/TypeScript project using Bun runtime that integrates with OpenCode's plugin system.

## Project Scope

### Core Capabilities

1. **Multi-Model Orchestration** - Dynamic model selection with fallback chains
2. **Parallel Background Agents** - Run multiple agents concurrently for increased throughput
3. **LSP/AST Tools** - Language Server Protocol integration and AST-based code analysis
4. **Plugin Architecture** - OpenCode plugin system with 76+ hook implementations
5. **CLI Interface** - Full command-line tool with install, run, doctor, and config commands

### Directory Structure

```
oh-my-openagent/
|-- bin/                    # CLI wrapper entry point (platform-aware binary launcher)
|-- src/
|   |-- cli/               # CLI implementation (22+ command modules)
|   |-- agents/           # Agent definitions (6 named agents + dynamic builder)
|   |-- hooks/            # Hook system (76+ implementations)
|   |-- features/         # Feature modules (22+ implementations)
|   |-- tools/            # Tool implementations (glob, grep, ast-grep, lsp, etc.)
|   |-- shared/           # Shared utilities (153 files)
|   |-- config/           # Configuration types and Zod schemas
|   |-- mcp/              # Model Context Protocol integration
|   |-- openclaw/         # OpenClaw integration
|-- packages/             # Platform-specific native binaries (11 platforms)
|-- tests/                # Integration tests
|-- docs/                 # Documentation
|-- script/               # Build and release scripts
```

## Architecture

### Entry Points

1. **CLI Entry** (`bin/oh-my-opencode.js`) - Platform-aware binary launcher
2. **CLI Program** (`src/cli/index.ts`) - Main CLI via Bun runtime
3. **Plugin Entry** (`src/index.ts`) - OpenCode plugin integration

### Key Patterns

| Pattern | Description |
|---------|-------------|
| Hook-Based Architecture | 76+ discrete hook implementations via `createHooks()` |
| Agent System | 6 named agents (librarian, oracle, momus, sisyphus, prometheus, hephaestus) with dynamic prompt builder |
| Feature Modules | 22+ feature implementations in `src/features/` |
| Tool Implementations | LSP, AST grep, glob, grep, vision, hashline-edit, interactive-bash, background tasks |
| Platform Packages | 11 platform-specific native binaries with AVX2 baseline variants |

### Configuration

- **Runtime**: Bun 1.x
- **Language**: TypeScript (strict mode)
- **Validation**: Zod v4 for runtime schema validation
- **Build**: `bun run build` compiles `src/index.ts` to `dist/`

## Key Dependencies

| Dependency | Purpose |
|------------|---------|
| `@opencode-ai/plugin` | OpenCode plugin system |
| `@opencode-ai/sdk` | OpenCode SDK |
| `@modelcontextprotocol/sdk` | MCP protocol |
| `@ast-grep/napi` | AST grep library |
| `commander` | CLI framework |
| `zod` | Schema validation |
| `picocolors` | Terminal colors |
| `picomatch` | Glob matching |

## Platform Support

Native binaries for 11 platform combinations:
- macOS: darwin-arm64, darwin-x64, darwin-x64-baseline
- Linux: linux-arm64, linux-arm64-musl, linux-x64, linux-x64-baseline, linux-x64-musl, linux-x64-musl-baseline
- Windows: windows-x64, windows-x64-baseline

## Build System

- **Build command**: `bun run build`
- **Binary build**: `bun run build:binaries` (via `script/build-binaries.ts`)
- **Schema build**: `bun run build:schema` (generates JSON schema)
- **Model capabilities**: `bun run build:model-capabilities`
