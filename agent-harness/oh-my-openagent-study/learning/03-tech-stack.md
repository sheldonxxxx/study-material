# Tech Stack: oh-my-openagent

## Language & Runtime

| Component | Technology |
|-----------|------------|
| **Language** | TypeScript 5.7.3+ (strict mode, ESNext) |
| **Runtime** | Bun 1.x |
| **Module System** | ESM (ESNext) |
| **Type Checking** | TypeScript compiler (tsc) |

### TypeScript Configuration

```json
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

**Notable choices:**
- Uses `bun-types` instead of `@types/node` for Bun-specific APIs
- `skipLibCheck: true` to avoid conflicts with Bun's native modules
- Bundler-style module resolution (matches Bun's behavior)

## Package Management

| Aspect | Choice |
|--------|--------|
| **Package Manager** | Bun (native workspaces) |
| **Lock File** | `bun.lock` |
| **Build Tool** | Bun build + TypeScript compiler |

**Why Bun:**
- Single tool replaces npm/yarn, tsc, jest, ts-node
- Native ESM support
- Fast execution and builds
- Used consistently across CI/CD (via `oven-sh/setup-bun@v2`)

**Monorepo approach:** Simple Bun workspaces (no Turborepo, Nx, or Lerna)

## Key Libraries

### Production Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| `@opencode-ai/sdk` | ^1.2.24 | OpenCode SDK integration |
| `@opencode-ai/plugin` | ^1.2.24 | OpenCode plugin interface |
| `@modelcontextprotocol/sdk` | ^1.25.2 | MCP (Model Context Protocol) |
| `zod` | ^4.1.8 | Schema validation for config |
| `@ast-grep/napi` | ^0.41.1 | AST analysis and transformation |
| `@ast-grep/cli` | ^0.41.1 | CLI for ast-grep |
| `commander` | ^14.0.2 | CLI argument parsing |
| `@clack/prompts` | ^0.11.0 | Terminal prompts UI |
| `picocolors` | ^1.1.1 | Terminal colors |
| `picomatch` | ^4.0.2 | Glob pattern matching |
| `diff` | ^8.0.3 | Text diffing |
| `js-yaml` | ^4.1.1 | YAML parsing |
| `jsonc-parser` | ^3.3.1 | JSONC parsing |
| `detect-libc` | ^2.0.0 | C library detection |
| `vscode-jsonrpc` | ^8.2.0 | JSON-RPC implementation |
| `@code-yeongyu/comment-checker` | ^0.7.0 | Comment quality checking |

### Dev Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| `typescript` | ^5.7.3 | TypeScript compiler |
| `bun-types` | 1.3.10 | Bun type definitions |
| `@types/js-yaml` | ^4.0.9 | YAML type definitions |
| `@types/picomatch` | ^3.0.2 | Glob type definitions |

## Build System

### Build Commands

| Command | Purpose |
|---------|---------|
| `bun run build` | Compile src/index.ts + src/cli/index.ts via Bun, emit declarations via tsc, generate schema |
| `bun run build:binaries` | Build 11 platform-specific compiled binaries |
| `bun run build:schema` | Generate JSON schema for plugin configuration |
| `bun run build:model-capabilities` | Generate model capability definitions |
| `bun run build:all` | Full build: build + binaries |
| `bun run typecheck` | Run `tsc --noEmit` |
| `bun run test` | Run Bun tests |

### Build Output

| Output | Location |
|--------|----------|
| Main package (ESM) | `dist/index.js` |
| Type declarations | `dist/index.d.ts` |
| CLI | `dist/cli/index.js` |
| Schema | `assets/oh-my-opencode.schema.json` |
| CLI entry | `bin/oh-my-opencode.js` |

### Binary Compilation

Uses `bun build --compile --minify --target=<platform>` to produce standalone executables.

**11 platform targets:**

| OS | CPU | Libc | Package |
|----|-----|------|---------|
| macOS | arm64 | - | oh-my-opencode-darwin-arm64 |
| macOS | x64 | - | oh-my-opencode-darwin-x64 |
| macOS | x64 (baseline) | - | oh-my-opencode-darwin-x64-baseline |
| Linux | x64 | glibc | oh-my-opencode-linux-x64 |
| Linux | x64 (baseline) | glibc | oh-my-opencode-linux-x64-baseline |
| Linux | arm64 | glibc | oh-my-opencode-linux-arm64 |
| Linux | x64 | musl | oh-my-opencode-linux-x64-musl |
| Linux | x64 (baseline) | musl | oh-my-opencode-linux-x64-musl-baseline |
| Linux | arm64 | musl | oh-my-opencode-linux-arm64-musl |
| Windows | x64 | - | oh-my-opencode-windows-x64 |
| Windows | x64 (baseline) | - | oh-my-opencode-windows-x64-baseline |

**Baseline builds** target older glibc/musl versions for compatibility.

## Project Structure

```
oh-my-openagent/
├── src/                          # Main TypeScript source (~1268 files)
│   ├── index.ts                  # Plugin entry point
│   ├── agents/                   # 11 agents (Sisyphus, Hephaestus, etc.)
│   ├── hooks/                    # 48 lifecycle hooks
│   ├── tools/                    # 26 tools
│   ├── features/                 # 19 feature modules
│   ├── shared/                   # 95+ utilities
│   ├── config/                   # Zod schema system
│   ├── cli/                      # CLI commands (Commander.js)
│   ├── mcp/                      # Built-in MCP servers
│   ├── plugin/                   # OpenCode hook handlers
│   └── plugin-handlers/          # 6-phase config loading pipeline
├── packages/                      # 11 platform binary packages
├── script/                        # Build and release scripts
├── bin/                           # CLI entry point
├── dist/                          # Build output
└── assets/                        # JSON schema
```

## Configuration Files

| File | Purpose |
|------|---------|
| `tsconfig.json` | TypeScript compiler options |
| `package.json` | Package definition, scripts, dependencies |
| `bunfig.toml` | Bun test preload configuration |
| `bun.lock` | Dependency lock file |

## Notable Technical Decisions

1. **Bun as unified toolchain** - Replaces npm/yarn, tsc, jest, ts-node
2. **No monorepo tooling** - Uses native Bun workspaces
3. **Platform-specific binaries** - 11 pre-compiled binaries for cross-platform distribution
4. **Baseline builds** - Separate targets for older glibc/musl versions
5. **Dual npm packages** - `oh-my-opencode` and `oh-my-openagent` (alias) both published
6. **Mock isolation** - Tests using `mock.module()` run in separate processes to prevent cache pollution
7. **Zod v4** - Using cutting-edge Zod with v4 schema syntax
8. **MCP integration** - First-class support for Model Context Protocol servers
