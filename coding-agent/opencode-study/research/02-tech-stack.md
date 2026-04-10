# OpenCode Tech Stack Analysis

## Overview

OpenCode is a **Bun-based monorepo** written primarily in TypeScript, with support for desktop applications (Tauri), web applications (SolidJS/SolidStart), and server-side rendering (Astro). It uses Turborepo for build orchestration, SST (v3) for cloud deployment, and Nix for development environment reproducibility.

## Language & Runtime

| Component | Language | Runtime |
|-----------|----------|---------|
| Core Application | TypeScript | Bun 1.3.11 |
| Desktop App | TypeScript | Tauri 2.x (Rust backend) |
| Web Frontend | TypeScript | Bun / Node 22+ |
| Server/Cloud Functions | TypeScript | Cloudflare Workers, Nitro |
| CI/CD | TypeScript (actions), Nix | Node 24, Bun |

- **Primary Package Manager**: Bun 1.3.11 (via `packageManager` in root `package.json`)
- **Node.js Versions**: 22+ (console, enterprise, function packages), 24 (CI deployment step)
- **Rust**: Stable toolchain for Tauri desktop builds

## Monorepo Structure

### Workspace Configuration

```json
{
  "workspaces": {
    "packages": [
      "packages/*",
      "packages/console/*",
      "packages/sdk/js",
      "packages/slack"
    ]
  }
}
```

### Key Packages

| Package | Purpose | Framework |
|---------|---------|-----------|
| `packages/opencode` | Core CLI application | Bun (Node-compatible) |
| `packages/app` | Main web application | SolidJS + Vite |
| `packages/desktop` | Tauri desktop wrapper | Tauri 2.x |
| `packages/desktop-electron` | Electron variant | Electron |
| `packages/ui` | Shared UI components | SolidJS |
| `packages/web` | Documentation site | Astro 5.x |
| `packages/console/app` | Admin console | SolidStart + Nitro |
| `packages/console/core` | Console backend | Hono |
| `packages/enterprise` | Enterprise features | SolidStart + Nitro |
| `packages/function` | Cloudflare Workers | Hono |
| `packages/sdk/js` | JavaScript SDK | TypeScript |
| `packages/util` | Shared utilities | TypeScript |
| `packages/identity` | Auth/identity service | Hono |
| `packages/slack` | Slack integration | TypeScript |
| `packages/storybook` | Component documentation | Storybook |
| `packages/containers/*` | Docker images | Multi-stage Docker |
| `packages/plugin` | Plugin system | TypeScript |
| `packages/script` | Build scripts | TypeScript |

## Build System

### Turborepo (v2.8.13)

Root `turbo.json` configures build orchestration:

```json
{
  "tasks": {
    "typecheck": {},
    "build": { "dependsOn": [], "outputs": ["dist/**"] },
    "opencode#test": { "dependsOn": ["^build"], "outputs": [] },
    "@opencode-ai/app#test": { "dependsOn": ["^build"], "outputs": [] }
  }
}
```

### Bun (Package Manager)

- **Lockfile**: `bunfig.toml` with `exact = true` for deterministic installs
- **Install Command**: `bun install` (with `--linker hoisted` on Windows)

### SST (v3.18.10) - Cloud Deployment

`sst.config.ts` defines deployment targets:

```typescript
export default $config({
  app(input) {
    return {
      name: "opencode",
      providers: {
        stripe: { apiKey: process.env.STRIPE_SECRET_KEY! },
        planetscale: "0.4.1",
      },
      home: "cloudflare",  // Cloudflare Workers as primary host
    }
  }
})
```

**Deployed Infrastructure:**
- Cloudflare (primary hosting)
- PlanetScale (MySQL database)
- Stripe (payments)

## Frontend Stack

### SolidJS Ecosystem

| Package | Version | Purpose |
|---------|---------|---------|
| `solid-js` | 1.9.10 | Core UI framework |
| `@solidjs/start` | `https://pkg.pr.new/@solidjs/start@dfb2020` | SSR framework |
| `@solidjs/router` | 0.15.4 | Routing |
| `@solidjs/meta` | 0.29.4 | SEO/meta tags |
| `@tanstack/solid-query` | 5.91.4 | Data fetching |
| `solid-list` | 0.3.0 | Virtual list |

### UI Component Libraries

- **@kobalte/core** (0.13.11) - Headless UI components
- **@opentui/core** + **@opentui/solid** (0.1.90) - OpenUI components
- **Motion** (12.34.5) - Animations
- **Tailwind CSS** (4.1.11) - Styling (via `@tailwindcss/vite`)

### Build Tools

| Tool | Version | Purpose |
|------|---------|---------|
| Vite | 7.1.4 | Build tool / dev server |
| `vite-plugin-solid` | 2.11.10 | SolidJS Vite plugin |
| `@tailwindcss/vite` | 4.1.11 | Tailwind integration |

### Documentation (Astro)

- **Astro** (5.7.13) - Static site generator
- **@astrojs/starlight** (0.34.3) - Documentation theme
- **Shiki** (3.20.0) + **marked-shiki** (1.2.1) - Syntax highlighting

## Backend Stack

### Web Framework

| Framework | Version | Purpose |
|-----------|---------|---------|
| Hono | 4.10.7 | Lightweight web framework |
| Nitro | 3.0.1-alpha.1 | Server framework |
| Astro | 5.7.13 | Static/dynamic sites |

### API & Validation

| Package | Version | Purpose |
|---------|---------|---------|
| Zod | 4.1.8 | Schema validation |
| `@hono/zod-validator` | 0.4.2 | Hono Zod middleware |
| `@hono/standard-validator` | 0.1.5 | OpenAPI validation |

### Database

| Package | Version | ORM |
|---------|---------|-----|
| Drizzle ORM | 1.0.0-beta.19-d95b7a4 | Type-safe SQL |
| Drizzle Kit | 1.0.0-beta.19-d95b7a4 | Migration tooling |
| PlanetScale | - | MySQL database (via drizzle) |

### Authentication

| Package | Version | Purpose |
|---------|---------|---------|
| `@openauthjs/openauth` | 0.0.0-20250322224806 | Auth framework |
| `jose` | 6.0.11 | JWT handling |
| `@octokit/rest` | 22.0.0 | GitHub API |
| `opencode-gitlab-auth` | 2.0.0 | GitLab OAuth |

## AI & LLM Integration

The opencode package includes extensive AI provider support:

### AI SDK Providers

| Provider | Package | Purpose |
|----------|---------|---------|
| Anthropic | `@ai-sdk/anthropic` | Claude models |
| OpenAI | `@ai-sdk/openai` | GPT models |
| Google | `@ai-sdk/google`, `@ai-sdk/google-vertex` | Gemini models |
| AWS Bedrock | `@ai-sdk/amazon-bedrock` | AWS-hosted models |
| Azure | `@ai-sdk/azure` | Azure OpenAI |
| Groq | `@ai-sdk/groq` | Fast inference |
| Perplexity | `@ai-sdk/perplexity` | Search models |
| TogetherAI | `@ai-sdk/togetherai` | Open models |
| Cerebras | `@ai-sdk/cerebras` | Fast inference |
| Cohere | `@ai-sdk/cohere` | Command models |
| XAI | `@ai-sdk/xai` | xAI models |
| DeepInfra | `@ai-sdk/deepinfra` | Open models |
| Vercel | `@ai-sdk/vercel` | Vercel AI |
| Gateway | `@ai-sdk/gateway` | AI gateway |

### AI Infrastructure

| Package | Version | Purpose |
|---------|---------|---------|
| `ai` | 5.0.124 | AI SDK core |
| `@ai-sdk/provider` | 2.0.1 | Provider interface |
| `ai-gateway-provider` | 2.3.1 | AI gateway |
| `partial-json` | 0.1.7 | Streaming JSON parsing |

### MCP & Protocol

| Package | Version | Purpose |
|---------|---------|---------|
| `@modelcontextprotocol/sdk` | 1.27.1 | MCP protocol |
| `@agentclientprotocol/sdk` | 0.14.1 | Agent protocol |

## Development Tools

### Code Quality

| Tool | Version | Purpose |
|------|---------|---------|
| TypeScript | 5.8.2 | Type checking |
| `@typescript/native-preview` | 7.0.0-dev.20251207.1 | Native type generation |
| Prettier | 3.6.2 | Code formatting |
| Husky | 9.1.7 | Git hooks |

### Effect Framework

| Package | Version | Purpose |
|---------|---------|---------|
| `effect` | 4.0.0-beta.35 | Functional programming |
| `@effect/platform-node` | 4.0.0-beta.35 | Node platform |
| `@effect/language-service` | 0.79.0 | TypeScript plugin |

### File Watching

| Package | Version | Purpose |
|---------|---------|---------|
| `@parcel/watcher` | 2.5.1 | Cross-platform file watching |
| `chokidar` | 4.0.3 | FS watching |

### Code Analysis

| Package | Version | Purpose |
|---------|---------|---------|
| `web-tree-sitter` | 0.25.10 | Tree-sitter bindings |
| `tree-sitter-bash` | 0.25.0 | Bash parsing |

## Desktop Development

### Tauri 2.x

| Package | Version | Purpose |
|---------|---------|---------|
| `@tauri-apps/api` | ^2 | Tauri JavaScript API |
| `@tauri-apps/cli` | ^2 | Build CLI |
| `@tauri-apps/plugin-*` | ~2 | Tauri plugins (clipboard, dialog, notification, shell, store, updater, etc.) |

### Desktop Plugins

- `@tauri-apps/plugin-clipboard-manager` - Clipboard access
- `@tauri-apps/plugin-deep-link` - URL handling
- `@tauri-apps/plugin-dialog` - Native dialogs
- `@tauri-apps/plugin-notification` - System notifications
- `@tauri-apps/plugin-os` - OS info
- `@tauri-apps/plugin-process` - Process management
- `@tauri-apps/plugin-shell` - Shell commands
- `@tauri-apps/plugin-store` - Persistent storage
- `@tauri-apps/plugin-updater` - Auto-updates
- `@tauri-apps/plugin-http` - HTTP requests
- `@tauri-apps/plugin-window-state` - Window state persistence

## Containerization

### Docker Images

| Base Image | Purpose |
|------------|---------|
| `ghcr.io/anomalyco/build/base:24.04` | Base build image |
| `ghcr.io/anomalyco/build/bun-node:24.04` | Bun + Node image |
| `ghcr.io/anomalyco/build/rust:24.04` | Rust toolchain |

### Multi-Stage Builds

- `packages/containers/base` - Alpine-based base
- `packages/containers/bun-node` - Node 24.4.0 + Bun 1.3.11
- `packages/containers/rust` - Adds Rust stable
- `packages/containers/tauri-linux` - Tauri build dependencies
- `packages/containers/publish` - Release publishing

## CI/CD

### GitHub Actions Workflows

| Workflow | Trigger | Purpose |
|----------|---------|---------|
| `deploy.yml` | push to dev/production | SST deployment |
| `containers.yml` | multi-arch builds | Docker multi-arch |
| `publish.yml` | releases | NPM publishing |
| `publish-vscode.yml` | releases | VSCode extension |
| `publish-github-action.yml` | releases | GitHub Action |
| `opencode.yml` | PR comments | AI review bot |
| `test.yml` | PRs | Test suite |
| `typecheck.yml` | PRs | Type checking |
| `storybook.yml` | PRs | Storybook deployment |
| `nix-eval.yml` | schedule | Nix cache refresh |

### Self-Hosted Runner

- `blacksmith-4vcpu-ubuntu-2404` - Used for opencode workflow

## Nix Integration

### flake.nix

Provides reproducible development environments:

```nix
devShells = {
  default = pkgs.mkShell {
    packages = with pkgs; [ bun nodejs_20 pkg-config openssl git ];
  };
};
```

Overlays for:
- `opencode` - Core application
- `opencode-desktop` - Desktop variant

## External Services

| Service | Integration |
|---------|-------------|
| Cloudflare | Hosting, Workers, KV |
| PlanetScale | MySQL database |
| Stripe | Payment processing |
| GitHub | API (Octokit), OAuth |
| GitLab | OAuth (opencode-gitlab-auth) |
| Vercel | AI SDK deployment |

## Key Architectural Decisions

1. **Bun-first**: Bun is the primary runtime and package manager
2. **Catalog versioning**: Shared dependency versions in root `package.json` catalog
3. **SolidJS for UI**: Reactive UI framework (not React)
4. **Cloudflare home**: Primary deployment target is Cloudflare Workers
5. **Effect for FP**: Uses Effect framework for type-safe functional programming
6. **Tauri for desktop**: Native desktop via Tauri (Rust backend)
7. **Multi-arch Docker**: Builds for both amd64 and arm64
8. **AI gateway**: Supports many AI providers with unified interface
