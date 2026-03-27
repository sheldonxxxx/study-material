# Dependencies: oh-my-openagent

## Philosophy

The project maintains a **lean dependency footprint** with a focus on:
- Minimal external dependencies
- Maximum use of Bun's built-in APIs
- Purpose-built internal solutions where appropriate

## Production Dependencies

### OpenCode Integration

| Package | Version | Purpose |
|---------|---------|---------|
| `@opencode-ai/sdk` | ^1.2.24 | SDK for interacting with OpenCode |
| `@opencode-ai/plugin` | ^1.2.24 | Plugin interface for OpenCode extensibility |

**Why:** Essential for the plugin to integrate with OpenCode. The project IS an OpenCode plugin.

### Model Context Protocol

| Package | Version | Purpose |
|---------|---------|---------|
| `@modelcontextprotocol/sdk` | ^1.25.2 | MCP client/server implementation |

**Why:** First-class MCP support for extensible tool servers. Allows skill-embedded MCPs that spin up on-demand.

### AST Analysis

| Package | Version | Purpose |
|---------|---------|---------|
| `@ast-grep/napi` | ^0.41.1 | Native AST analysis and transformation |
| `@ast-grep/cli` | ^0.41.1 | CLI for ast-grep |

**Why:** AST-aware code search and rewriting across 25 languages. Enables IDE-precision refactoring tools for agents.

**Trusted dependency:** Declared in `trustedDependencies` for security during install.

### Configuration & Validation

| Package | Version | Purpose |
|---------|---------|---------|
| `zod` | ^4.1.8 | Schema validation for plugin configuration |

**Why:** Type-safe configuration validation with generated TypeScript types. Using v4 for cutting-edge syntax.

### CLI Infrastructure

| Package | Version | Purpose |
|---------|---------|---------|
| `commander` | ^14.0.2 | CLI argument parsing |
| `@clack/prompts` | ^0.11.0 | Terminal prompts UI |
| `picocolors` | ^1.1.1 | Terminal colors for output |
| `picomatch` | ^4.0.2 | Glob pattern matching |

**Why:**
- Commander: Standard CLI argument parsing
- Clack: Beautiful terminal prompts (progress, spinners, cancellation)
- Picocolors: Lightweight terminal colors
- Picomatch: Fast glob matching for file patterns

### Code Processing

| Package | Version | Purpose |
|---------|---------|---------|
| `diff` | ^8.0.3 | Text diffing for file changes |
| `js-yaml` | ^4.1.1 | YAML parsing for config files |
| `jsonc-parser` | ^3.3.1 | JSONC parsing (JSON with comments) |

**Why:**
- Diff: Visual diff output for agent edits
- YAML: Config file support
- JSONC: Comment support in JSON config files

### Platform & System

| Package | Version | Purpose |
|---------|---------|---------|
| `detect-libc` | ^2.0.0 | Detect C library version for binary compatibility |

**Why:** Determines which platform binary to use based on system libc version (glibc vs musl).

### Communication

| Package | Version | Purpose |
|---------|---------|---------|
| `vscode-jsonrpc` | ^8.2.0 | JSON-RPC implementation |

**Why:** LSP communication for IDE-like features (rename, goto, references, diagnostics).

### Custom Internal Package

| Package | Version | Purpose |
|---------|---------|---------|
| `@code-yeongyu/comment-checker` | ^0.7.0 | Comment quality enforcement |

**Why:** Custom linter to prevent AI-generated comment slop. Ensures code comments are meaningful.

**Trusted dependency:** Declared in `trustedDependencies` for native module security.

## Dev Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| `typescript` | ^5.7.3 | TypeScript compiler |
| `bun-types` | 1.3.10 | Bun-specific type definitions |
| `@types/js-yaml` | ^4.0.9 | YAML type definitions |
| `@types/picomatch` | ^3.0.2 | Glob type definitions |

**Why bun-types over @types/node:** Bun has its own runtime APIs (Bun.file, Bun.serve, etc.) that require bun-specific types.

## Optional Dependencies

### Platform Binaries (11 packages)

```json
{
  "oh-my-opencode-darwin-arm64": "3.11.0",
  "oh-my-opencode-darwin-x64": "3.11.0",
  "oh-my-opencode-darwin-x64-baseline": "3.11.0",
  "oh-my-opencode-linux-arm64": "3.11.0",
  "oh-my-opencode-linux-arm64-musl": "3.11.0",
  "oh-my-opencode-linux-x64": "3.11.0",
  "oh-my-opencode-linux-x64-baseline": "3.11.0",
  "oh-my-opencode-linux-x64-musl": "3.11.0",
  "oh-my-opencode-linux-x64-musl-baseline": "3.11.0",
  "oh-my-opencode-windows-x64": "3.11.0",
  "oh-my-opencode-windows-x64-baseline": "3.11.0"
}
```

**Why optional:** Platform binaries are installed based on the user's system, not required for all installations.

## Dependency Graph Summary

```
@opencode-ai/sdk ─────────────────────────────────┐
@opencode-ai/plugin ──────────────────────────────┼── OpenCode integration
@modelcontextprotocol/sdk ────────────────────────┼── MCP support
                                                    │
zod ───────────────────────────────────────────────┤── Config validation
                                                    │
@ast-grep/napi ───────────────────────────────────┤
@ast-grep/cli ────────────────────────────────────┼── AST analysis
                                                    │
commander ─────────────────────────────────────────┤
@clack/prompts ───────────────────────────────────┤── CLI tools
picocolors ────────────────────────────────────────┤
picomatch ────────────────────────────────────────┤
                                                    │
diff ──────────────────────────────────────────────┤
js-yaml ───────────────────────────────────────────┼── Code processing
jsonc-parser ──────────────────────────────────────┤
                                                    │
detect-libc ───────────────────────────────────────┤── Platform detection
vscode-jsonrpc ────────────────────────────────────┼── LSP communication
                                                    │
@code-yeongyu/comment-checker ─────────────────────┴── Comment quality
```

## Dependency Count

| Category | Count |
|----------|-------|
| Production dependencies | 16 |
| Dev dependencies | 4 |
| Optional platform packages | 11 |
| **Total** | 31 |

## Why These Libraries (Not Others)

| Not Used | Reason |
|----------|--------|
| ESLint / Prettier | AST-Grep handles linting/formatting |
| Jest / Vitest | Bun test is sufficient |
| Lodash | Bun runtime APIs cover most needs |
| Axios / Fetch | Bun's built-in fetch |
| Express / Fastify | Not a web server |
| TypeORM / Prisma | Not a database ORM |
| Next.js / Remix | Not a web app |
| Webpack / Rollup / Vite | Bun build handles this |
| Turborepo / Nx / Lerna | Native Bun workspaces sufficient |

## Dependency Security

### Trusted Dependencies

Three packages with native modules are marked as trusted:
- `@ast-grep/cli`
- `@ast-grep/napi`
- `@code-yeongyu/comment-checker`

These can run post-install scripts during `bun install`.

### npm Provenance

All npm publishes use OIDC provenance for supply chain security:
```bash
npm publish --access public --provenance
```
