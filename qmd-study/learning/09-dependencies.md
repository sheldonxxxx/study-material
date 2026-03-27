# QMD Dependencies

## Dependency Overview

Total dependencies: ~15 (7 runtime, 6 optional platform-specific, 3 dev)

## Key Dependencies Analysis

### Search Infrastructure

| Dependency | Purpose | Health |
|------------|---------|--------|
| `better-sqlite3` ^12.4.5 | Synchronous SQLite bindings | Stable, actively maintained |
| `sqlite-vec` ^0.1.7-alpha.2 | Vector search extension | Active development (alpha) |
| `node-llama-cpp` ^3.17.1 | Local LLM inference | Active, popular for GGUF |

### Protocol

| Dependency | Purpose | Health |
|------------|---------|--------|
| `@modelcontextprotocol/sdk` ^1.25.1 | MCP server | Official MCP Foundation SDK |

### Utilities

| Dependency | Purpose | Health |
|------------|---------|--------|
| `fast-glob` ^3.3.0 | File globbing | Stable, widely used |
| `picomatch` ^4.0.0 | Glob matching | Stable |
| `yaml` ^2.8.2 | Config parsing | Stable |
| `zod` ^4.2.1 | Validation | Active development (v4.x) |

## Notable Dependency Choices

### better-sqlite3 over sql.js or prisma

Chosen for synchronous SQLite access with native performance. No ORM layer.

### sqlite-vec over pgvector or Milvus

SQLite-native vector search keeps everything local and simple. No separate database server.

### node-llama-cpp over Ollama or API-based

Enables fully local LLM inference without external services. GGUF models run directly.

### Zod v4 over typical validation libraries

Latest Zod version for runtime TypeScript validation.

## Dependency Health Practices

### Lockfile Management

- **Bun**: `--frozen-lockfile` enforced in CI (since v2.0.1 fix for #386)
- **npm**: Uses `package-lock.json` with npm

### Recent Dependency Updates

From CHANGELOG:
- `better-sqlite3` 11.x to 12.x (sync in v2.0.1)
- Node 25 support added
- Zod v4 adoption

### Platform-Specific Binaries

Optional dependencies provide pre-built `sqlite-vec` for:
- macOS arm64 and x64
- Linux arm64 and x64
- Windows x64

No native compilation required for most users.

## Dependency Vulnerability Considerations

- Native modules (`better-sqlite3`, `node-llama-cpp`) require platform-specific binaries
- node-llama-cpp uses native bindings for CUDA/Metal acceleration when available
- sqlite-vec is alpha software but widely used in production

## Summary

| Aspect | Assessment |
|--------|------------|
| Dependency count | Low (~15 total) |
| Dependency health | Good (stable, active projects) |
| Lockfile discipline | Strict (frozen lockfile enforced) |
| Native modules | Yes (better-sqlite3, node-llama-cpp, sqlite-vec) |
| Alpha dependencies | sqlite-vec (intentional, for vector search) |
| Update frequency | Moderate (recent v2.0.x stabilizations) |
