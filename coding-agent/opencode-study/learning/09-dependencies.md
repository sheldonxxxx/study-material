# Dependencies

## Catalog System

OpenCode uses a **monorepo catalog** for centralized dependency management. The catalog is defined in the root `package.json` under `catalog` and referenced across all workspace packages.

### How Catalog Works

```json
{
  "catalog": {
    "typescript": "5.8.2",
    "hono": "4.10.7",
    "solid-js": "1.9.10"
  }
}
```

Packages reference catalog entries:

```json
{
  "devDependencies": {
    "typescript": "catalog:"  // Resolves to 5.8.2
  }
}
```

This ensures consistent versions across the monorepo.

## Key Dependencies

### AI / LLM (20+ providers)

| Package | Version | Purpose |
|---------|---------|---------|
| `ai` | 5.0.124 | Core AI SDK |
| `@ai-sdk/anthropic` | 2.0.65 | Claude |
| `@ai-sdk/openai` | 2.0.89 | GPT-4 |
| `@ai-sdk/google` | 2.0.54 | Gemini |
| `@ai-sdk/amazon-bedrock` | 3.0.82 | AWS models |
| `@ai-sdk/azure` | 2.0.91 | Azure OpenAI |
| `@ai-sdk/cohere` | 2.0.22 | Command |
| `@ai-sdk/groq` | 2.0.34 | Groq |
| `@ai-sdk/xai` | 2.0.51 | Grok |
| `@ai-sdk/cerebras` | 1.0.36 | Cerebras |
| `@ai-sdk/togetherai` | 1.0.34 | Together |
| `@ai-sdk/vercel` | 1.0.33 | Vercel AI |
| `@ai-sdk/deepinfra` | 1.0.36 | DeepInfra |
| `@ai-sdk/mistral` | 2.0.27 | Mistral |
| `@ai-sdk/perplexity` | 2.0.23 | Perplexity |
| `@ai-sdk/openai-compatible` | 1.0.32 | OpenAI-compatible |
| `@ai-sdk/gateway` | 2.0.30 | AI Gateway |
| `@ai-sdk/google-vertex` | 3.0.106 | Google Vertex |
| `@openrouter/ai-sdk-provider` | 1.5.4 | OpenRouter |
| `ai-gateway-provider` | 2.3.1 | AI Gateway |
| `gitlab-ai-provider` | 5.3.3 | GitLab AI |

### Web Framework

| Package | Version | Purpose |
|---------|---------|---------|
| `hono` | 4.10.7 | API server |
| `hono-openapi` | 1.1.2 | OpenAPI docs |
| `@hono/zod-validator` | 0.4.2 | Zod middleware |

### UI

| Package | Version | Purpose |
|---------|---------|---------|
| `solid-js` | 1.9.10 | UI framework |
| `@solidjs/router` | 0.15.4 | Router |
| `@solidjs/meta` | 0.29.4 | Meta tags |
| `@solidjs/start` | (prerelease) | Full-stack framework |
| `@opentui/core` | 0.1.90 | TUI components |
| `@opentui/solid` | 0.1.90 | SolidJS bindings |
| `@kobalte/core` | 0.13.11 | Accessible components |
| `tailwindcss` | 4.1.11 | Styling |
| `@tailwindcss/vite` | 4.1.11 | Vite plugin |

### Database

| Package | Version | Purpose |
|---------|---------|---------|
| `drizzle-orm` | 1.0.0-beta.19-d95b7a4 | ORM |
| `drizzle-kit` | 1.0.0-beta.19-d95b7a4 | Migrations |

### Effect Ecosystem

| Package | Version | Purpose |
|---------|---------|---------|
| `effect` | 4.0.0-beta.35 | FP |
| `@effect/platform-node` | 4.0.0-beta.35 | Platform |
| `@effect/language-service` | 0.79.0 | LSP |

### Utilities

| Package | Version | Purpose |
|---------|---------|---------|
| `zod` | 4.1.8 | Validation |
| `remeda` | 2.26.0 | FP utils |
| `shiki` | 3.20.0 | Syntax highlighting |
| `marked` | 17.0.1 | Markdown |
| `marked-shiki` | 1.2.1 | MD + highlighting |
| `diff` | 8.0.2 | Text diffing |
| `fuzzysort` | 3.1.0 | Fuzzy search |
| `ulid` | 3.0.1 | IDs |
| `semver` | ^7.6.3 | Version parsing |

### Code Intelligence

| Package | Version | Purpose |
|---------|---------|---------|
| `tree-sitter` | (via web-tree-sitter) | Parsing |
| `web-tree-sitter` | 0.25.10 | WASM parser |
| `tree-sitter-bash` | 0.25.0 | Bash grammar |
| `vscode-jsonrpc` | 8.2.1 | LSP transport |
| `vscode-languageserver-types` | 3.17.5 | LSP types |

### Desktop

| Package | Version | Purpose |
|---------|---------|---------|
| `@tauri-apps/api` | ^2 | Tauri JS API |
| `electron` | (trusted) | Electron |
| `electron-builder` | (in desktop-electron) | Packaging |

### External Integrations

| Package | Version | Purpose |
|---------|---------|---------|
| `@actions/core` | 1.11.1 | GitHub Actions |
| `@actions/github` | 6.0.1 | GitHub API |
| `@octokit/rest` | 22.0.0 | REST API |
| `@octokit/graphql` | 9.0.2 | GraphQL API |
| `@modelcontextprotocol/sdk` | 1.27.1 | MCP |
| `@agentclientprotocol/sdk` | 0.14.1 | Agent protocol |
| `@aws-sdk/client-s3` | 3.933.0 | S3 |
| `google-auth-library` | 10.5.0 | Google auth |
| `@openauthjs/openauth` | 0.0.0-20250322224806 | Auth |

### Dev Tools

| Package | Version | Purpose |
|---------|---------|---------|
| `turbo` | 2.8.13 | Build orchestration |
| `typescript` | 5.8.2 | Type checking |
| `@typescript/native-preview` | 7.0.0-dev.20251207.1 | Native TS |
| `prettier` | 3.6.2 | Formatting |
| `husky` | 9.1.7 | Git hooks |
| `@playwright/test` | 1.51.0 | E2E testing |

## Versioning Strategy

### Semantic Versioning

OpenCode follows semver for releases:
- `major`: Breaking changes
- `minor`: New features (backward compatible)
- `patch`: Bug fixes

### Catalog vs Fixed Versions

- **Catalog entries**: Used for shared dependencies to ensure consistency
- **Fixed versions**: Used for packages with breaking changes or specific needs

### Beta Releases

- Beta channel builds from `beta` branch
- Releases to separate `opencode-beta` repository
- Used for testing before production

### Snapshot Releases

- `snapshot-*` branches trigger snapshot builds
- For bleeding-edge testing

## Dependency Updates

### Automated

- `dependabot` or similar for safety updates
- Security patches tracked separately

### Manual

- `./script/version.ts` for version bumping
- Version job in `publish.yml` determines next version

## Trusted Dependencies

Some packages are marked as trusted (for npm script execution):

```json
{
  "trustedDependencies": [
    "esbuild",
    "protobufjs",
    "tree-sitter",
    "tree-sitter-bash",
    "web-tree-sitter",
    "electron"
  ]
}
```

## Patched Dependencies

Some dependencies have local patches applied:

```json
{
  "patchedDependencies": {
    "@standard-community/standard-openapi@0.2.9": "patches/@standard-community%2Fstandard-openapi@0.2.9.patch",
    "@openrouter/ai-sdk-provider@1.5.4": "patches/@openrouter%2Fai-sdk-provider@1.5.4.patch",
    "@ai-sdk/xai@2.0.51": "patches/@ai-sdk%2Fxai@2.0.51.patch",
    "solid-js@1.9.10": "patches/solid-js@1.9.10.patch"
  }
}
```

## Workspace Dependencies

Internal packages reference each other via `workspace:*`:

```json
{
  "@opencode-ai/sdk": "workspace:*",
  "@opencode-ai/plugin": "workspace:*",
  "@opencode-ai/script": "workspace:*",
  "@opencode-ai/util": "workspace:*"
}
```

## Build-time Dependencies

### Root Package.json

```json
{
  "devDependencies": {
    "@actions/artifact": "5.0.1",
    "@tsconfig/bun": "catalog:",
    "@typescript/native-preview": "catalog:",
    "glob": "13.0.5",
    "husky": "9.1.7",
    "prettier": "3.6.2",
    "semver": "^7.6.0",
    "sst": "3.18.10",
    "turbo": "2.8.13"
  }
}
```

## External Tooling

### Tauri

- Rust-based desktop framework
- Requires Rust toolchain for desktop builds
- Tauri CLI cached in CI via `taiki-e/cache-cargo-install-action`

### Bun

- Primary runtime and package manager
- Version locked: 1.3.11
- Native SQLite support via `Bun.file()`
