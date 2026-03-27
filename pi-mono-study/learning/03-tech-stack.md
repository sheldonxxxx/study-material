# Tech Stack

## Languages and Runtimes

| Component | Technology | Version |
|-----------|------------|---------|
| Primary Language | TypeScript | 5.9.2 (root), 5.7.3 (packages) |
| Target | ES2022 | Module system: Node16 |
| Runtime | Node.js | >=20.0.0 (CI uses Node 22) |
| Binary Builder | Bun | 1.2.20 |
| Scripting | tsx | 4.20.3 |

## Monorepo Tooling

**No Turborepo, Nx, or Lerna** -- pi-mono uses native npm workspaces.

```json
"workspaces": [
  "packages/*",
  "packages/web-ui/example",
  "packages/coding-agent/examples/extensions/*"
]
```

### TypeScript Configuration

- Base config: `tsconfig.base.json` (shared across all packages)
- Package configs: Each package has its own `tsconfig.build.json` extending the base
- Compiler: `tsgo` (tsx-based TypeScript compiler wrapper)
- Module resolution: Node16
- Decorator support: Enabled (`experimentalDecorators`, `emitDecoratorMetadata`)

### Code Quality Tools

| Tool | Purpose |
|------|---------|
| **Biome** | Linting + Formatting (biome.json, v2.3.5) |
| **TypeScript** | Type checking via `tsc --noEmit` |
| **Husky** | Git hooks via `prepare` script |

## Packages in Monorepo

| Package | Version | Purpose |
|---------|---------|---------|
| `@mariozechner/pi-ai` | 0.62.0 | Unified multi-provider LLM API |
| `@mariozechner/pi-agent-core` | 0.62.0 | Agent runtime with tool calling |
| `@mariozechner/pi-coding-agent` | 0.62.0 | Interactive coding agent CLI (`pi`) |
| `@mariozechner/pi-mom` | 0.62.0 | Slack bot delegating to pi |
| `@mariozechner/pi-tui` | 0.62.0 | Terminal UI library |
| `@mariozechner/pi-web-ui` | 0.62.0 | Lit web components for AI chat |
| `@mariozechner/pi-pods` | 0.62.0 | CLI for vLLM GPU pod deployments |

### Layered Architecture

```
pi-ai (LLM abstraction)
    |
pi-agent-core (agent runtime)
    |
pi-coding-agent (CLI) --> pi-tui (shared terminal UI)
                   |
               pi-web-ui (web components)
```

## Build System

| Command | Purpose |
|---------|---------|
| `npm run clean` | Remove all dist folders |
| `npm run build` | Sequential build: tui, ai, agent, coding-agent, mom, web-ui, pods |
| `npm run dev` | Concurrent dev mode for all packages |
| `npm run check` | Biome lint/format + TypeScript check + browser smoke test |
| `npm run test` | Run tests across all workspaces |

### Binary Compilation

Uses **Bun** to compile the coding-agent into standalone binaries for:
- macOS: arm64, x64
- Linux: x64, arm64
- Windows: x64

## Testing

| Package | Test Runner |
|---------|-------------|
| tui | Node.js built-in test (`node --test`) |
| ai, agent, coding-agent | vitest |

## Key Dependencies

| Dependency | Purpose |
|------------|---------|
| `@anthropic-ai/sdk` | Anthropic LLM provider |
| `@google/genai` | Google Gemini provider |
| `@mistralai/mistralai` | Mistral provider |
| `openai` | OpenAI provider |
| `@aws-sdk/client-bedrock-runtime` | AWS Bedrock provider |
| `@sinclair/typebox` | JSON validation |
| `partial-json` | Streaming JSON parsing |
| `lit` | Web components (peer dependency) |
| `koffi` | Optional native binding for VT input |
| `@silvia-odwyer/photon-node` | Rust-based image processing |
| `@anthropic-ai/sandbox-runtime` | Sandbox execution for mom bot |

## Dependency Overrides

```json
"overrides": {
  "rimraf": "6.1.2",
  "fast-xml-parser": "5.3.8",
  "gaxios": { "rimraf": "6.1.2" }
}
```
