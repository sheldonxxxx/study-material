# Dependencies

## Direct Dependencies (package.json)

### Production Dependencies

| Dependency | Version | Purpose |
|------------|---------|---------|
| @lancedb/lancedb | ^0.26.2 | LanceDB vector database for memory storage |
| @sinclair/typebox | 0.34.48 | JSON schema TypeScript types for runtime validation |
| apache-arrow | 18.1.0 | Columnar data format for LanceDB interop |
| json5 | ^2.2.3 | JSON5 parser for config files |
| openai | ^6.21.0 | OpenAI API client for embeddings and LLM calls |
| proper-lockfile | ^4.1.2 | File locking for cross-process safety |

### Dev Dependencies

| Dependency | Version | Purpose |
|------------|---------|---------|
| commander | ^14.0.0 | CLI argument parsing |
| jiti | ^2.6.0 | TypeScript/ESM execution without compilation |
| typescript | ^5.9.3 | Type checking only (not used for build) |

## Transitive Dependencies

The dependency tree is remarkably shallow with only ~25 unique packages in the lockfile.

### From @lancedb/lancedb (0.26.2)

- `reflect-metadata` (^0.2.2) -- Apache-2.0
- Platform-specific optional deps: `@lancedb/lancedb-darwin-arm64`, `@lancedb/lancedb-linux-x64-gnu`, `@lancedb/lancedb-linux-x64-musl`, `@lancedb/lancedb-linux-arm64-gnu`, `@lancedb/lancedb-linux-arm64-musl`, `@lancedb/lancedb-win32-x64-msvc`, `@lancedb/lancedb-win32-arm64-msvc`
- **Peer dep:** `apache-arrow` >=15.0.0 <=18.1.0 (already satisfied)

### From apache-arrow (18.1.0)

- `@swc/helpers` (^0.5.11) -- Apache-2.0, pulls in `tslib`
- `@types/command-line-args` (^5.2.3) -- MIT
- `@types/command-line-usage` (^5.0.4) -- MIT
- `@types/node` (^20.13.0) -- MIT, pulls in `undici-types`
- `command-line-args` (^5.2.1) -- MIT
- `command-line-usage` (^7.0.1) -- MIT
- `flatbuffers` (^24.3.25) -- Apache-2.0
- `json-bignum` (^0.0.3)
- `tslib` (^2.6.2) -- 0BSD

### From openai (6.22.0)

- **No required transitive dependencies**
- Optional peer deps: `ws` (^8.18.0), `zod` (^3.25 or ^4.0) -- both optional

### From proper-lockfile (4.1.2)

- `graceful-fs` (^4.2.4) -- ISC
- `retry` (^0.12.0) -- MIT
- `signal-exit` (^3.0.2) -- ISC

### From json5 (2.2.3)

- No transitive dependencies

## Dependency Health Assessment

### Version Constraints

| Dependency | Constraint | Resolved | Assessment |
|------------|------------|----------|------------|
| @lancedb/lancedb | ^0.26.2 | 0.26.2 | Loose caret -- allows minor/patch updates |
| @sinclair/typebox | 0.34.48 | 0.34.48 | **Exact version** -- unusual, may indicate compatibility requirement |
| apache-arrow | 18.1.0 | 18.1.0 | **Exact version** -- unusual, likely for LanceDB peer dep compatibility |
| json5 | ^2.2.3 | 2.2.3 | Loose caret |
| openai | ^6.21.0 | 6.22.0 | Loose caret, resolved to 6.22.0 |
| proper-lockfile | ^4.1.2 | 4.1.2 | Loose caret |
| commander | ^14.0.0 | 14.0.3 | Loose caret |
| jiti | ^2.6.0 | 2.6.1 | Loose caret |
| typescript | ^5.9.3 | 5.9.3 | Loose caret |

### Notable Observations

1. **Exact version pins on apache-arrow and typebox** -- both at hardcoded versions rather than semver ranges. The apache-arrow pin aligns with LanceDB's peer dependency constraint (`>=15.0.0 <=18.1.0`). The typebox pin may indicate a specific API compatibility requirement.

2. **LanceDB is the heaviest dependency** -- brings in platform-specific native binaries (~7 optional platform variants) and apache-arrow. This is expected for a vector database plugin.

3. **openai SDK has no transitive deps** -- the openai package is self-contained, only declaring optional peer deps for `ws` and `zod`.

4. **No security-sensitive transitive deps identified** -- the dependency tree is small and composed of well-known packages. `graceful-fs`, `retry`, `signal-exit` are all mature, low-risk utilities.

5. **No known vulnerable dependencies** -- at time of analysis, no known CVEs in the transitive dependency tree.

6. **Lockfile vs package.json version mismatch** -- lockfile shows v1.1.0-beta.9, package.json shows v1.1.0-beta.10. The lockfile is stale.

## No Internal Workspace Dependencies

The project is a single package with no internal workspace dependencies (`../packages/*` or similar). All code is in the root package.

## Ecosystem Dependencies

The plugin integrates with (via API, not npm deps):
- OpenAI API
- Jina AI (embeddings + reranking)
- Azure OpenAI
- Any OpenAI-compatible API endpoint

## Summary Table

| Dimension | Assessment |
|-----------|------------|
| Dependency count | Very low (~25 unique packages) |
| Native binary risk | Medium -- LanceDB platform variants (normal for vector DB) |
| Security concerns | None identified |
| Maintenance burden | Low -- shallow tree, well-known packages |
| Lockfile freshness | **Stale** -- lockfile beta.9 vs package.json beta.10 |
