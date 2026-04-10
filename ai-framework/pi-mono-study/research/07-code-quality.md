# pi-mono Code Quality Assessment

## Executive Summary

The pi-mono project demonstrates solid engineering practices with comprehensive testing, strict TypeScript configuration, and consistent tooling. The codebase shows maturity in error handling, but has some areas for improvement in code review patterns and logging strategies.

**Overall Quality: GOOD**

---

## 1. Testing Coverage and Quality

### Test Infrastructure

| Package | Test Runner | Test Count |
|---------|------------|------------|
| `@mariozechner/pi-ai` | vitest | ~40 test files |
| `@mariozechner/pi-agent-core` | vitest | ~4 test files |
| `@mariozechner/pi-coding-agent` | vitest | ~60 test files |
| `@mariozechner/pi-tui` | Node.js built-in test | ~22 test files |

### Test Coverage Assessment

**Strengths:**

- **Provider-level abort testing**: `abort.test.ts` tests cancellation across all LLM providers (Google, OpenAI, Anthropic, Azure, Mistral, Bedrock, etc.) with retry configuration
- **Tool-level unit tests**: Extensive tests for read/write/edit/find/grep tools with edge cases (truncation, offset/limit, image detection via magic bytes)
- **Session management tests**: Agent session compaction, branching, retry, and concurrent operation tests
- **Integration fixtures**: Test data in `fixtures/` directory (JSONL session files, PNG images)
- **Mock stream patterns**: `MockAssistantStream` class in agent tests for controlled testing

**Patterns Observed:**

```typescript
// vitest with retry for flaky provider tests
it("should abort mid-stream", { retry: 3 }, async () => { ... });

// Environment-gated tests for paid providers
describe.skipIf(!process.env.ANTHROPIC_OAUTH_TOKEN)("Anthropic Provider Abort", () => { ... });

// Node.js built-in test runner (tui package)
import { describe, it } from "node:test";
```

**Coverage Gaps:**

- No visible unit tests for `pi-mom` (Slack bot) package
- No visible unit tests for `pi-pods` (GPU pod CLI) package
- Limited browser/E2E testing visible (only `check:browser-smoke` script)
- Web UI component testing not evident

### Test File Patterns

```typescript
// Test utility patterns - test-harness.ts provides faux provider for session testing
// AgentSession test harness with faux provider for local testing without API calls
class MockAssistantStream extends EventStream<AssistantMessageEvent, AssistantMessage> {
    constructor() {
        super(
            (event) => event.type === "done" || event.type === "error",
            (event) => {
                if (event.type === "done") return event.message;
                if (event.type === "error") return event.error;
                throw new Error("Unexpected event type");
            },
        );
    }
}
```

---

## 2. Type System and TypeScript Configuration

### Base Configuration (`tsconfig.base.json`)

```json
{
    "compilerOptions": {
        "target": "ES2022",
        "module": "Node16",
        "strict": true,
        "declaration": true,
        "declarationMap": true,
        "sourceMap": true,
        "moduleResolution": "Node16",
        "experimentalDecorators": true,
        "emitDecoratorMetadata": true
    }
}
```

### Strict Mode Analysis

**Enabled:** `strict: true` (includes `strictNullChecks`, `noImplicitAny`, `strictFunctionTypes`, etc.)

**Explicit `any` Suppressions (biome.json):**
```json
"suspicious": {
    "noExplicitAny": "off",
    "noControlCharactersInRegex": "off",
    "noEmptyInterface": "off"
}
```

The project uses `any` in specific contexts:
- API key resolution (`getEnvApiKey` uses `any` for provider argument)
- Test file content extraction (`getTextOutput(result: any)`)

### Declaration Generation

**Package Exports with .d.ts:**
- Each package generates `.d.ts` files via `declaration: true`
- Subpath exports explicitly define types (e.g., `"./oauth": { "types": "./dist/oauth.d.ts" }`)
- Notable: `bedrock-provider.d.ts` exists as standalone type declaration

**Build Output Verification:**
```bash
# Package tsconfig.build.json excludes .d.ts from source
"exclude": ["node_modules", "dist", "**/*.d.ts", "src/**/*.d.ts"]
```

---

## 3. Biome Linting and Formatting

### Configuration (`biome.json`)

```json
{
    "linter": {
        "enabled": true,
        "rules": {
            "recommended": true,
            "style": {
                "noNonNullAssertion": "off",
                "useConst": "error",
                "useNodejsImportProtocol": "off"
            }
        }
    },
    "formatter": {
        "indentStyle": "tab",
        "indentWidth": 3,
        "lineWidth": 120
    }
}
```

### Key Observations

- **Tab indentation**: 3-tab indent with 120 character line width (deviation from common 80)
- **Biome in CI**: `npm run check` runs `biome check --write --error-on-warnings`
- **Recommended rules enabled**: Standard Biome recommended set
- **Selective disabling**: `noNonNullAssertion` and `noExplicitAny` explicitly disabled

### File Coverage

```json
"files": {
    "includes": [
        "packages/*/src/**/*.ts",
        "packages/*/test/**/*.ts",
        "packages/coding-agent/examples/**/*.ts",
        "packages/web-ui/src/**/*.ts"
    ],
    "excludes": [
        "!**/node_modules/**/*",
        "!**/test-sessions.ts",
        "!**/models.generated.ts",
        "!packages/web-ui/src/app.css",
        "!packages/mom/data/**/*"
    ]
}
```

---

## 4. Commit History and Code Review Patterns

### Commit Message Convention

**Format:** `type(scope): message` (Conventional Commits)

**Recent examples:**
```
fix(tui): reset viewport state after shrink
fix(coding-agent): add follow-up docs, changelog, and precedence tests closes #2429
feat(coding-agent): Add sessionDir support in settings.json (#2598)
chore(coding-agent): emit startup benchmark metrics
test(tui): remove stale slash autocomplete chaining case
docs: complete changelog entries for unreleased changes
```

### Patterns Observed

**Strengths:**
- Consistent type prefixes: `fix:`, `feat:`, `chore:`, `test:`, `docs:`
- Scope often references package name
- Issue references in commit messages (`closes #2429`)
- Separate test commits for stabilization

**Observations:**
- Very frequent small commits (often 1-3 per day)
- Mix of fix-only commits and combo commits (fix+test+docs)
- No visible PR review process in commit messages (commits directly to main)
- Changelog maintenance as part of development workflow

### GitHub Actions Integration

- `ci.yml`: Build, check, test on Node 22 for push/PR
- `build-binaries.yml`: Cross-platform binary compilation
- `pr-gate.yml`: Auto-close PRs from non-approved contributors

---

## 5. Error Handling Patterns

### Strategy Overview

**1. Synchronous Error Throwing:**
```typescript
// Direct Error throwing for invariant violations
if (!provider) {
    throw new Error(`No API provider registered for api: ${api}`);
}
```

**2. Try-Catch with Error Handling:**
```typescript
try {
    const response = await fetch(url);
    const data = await response.json();
} catch (error) {
    if (error instanceof Error) {
        return { error: error.message };
    }
    return { error: String(error) };
}
```

**3. Result Pattern (less common):**
```typescript
// Pattern used in validation.ts
try {
    // validation logic
} catch (_e) {
    // Errors silenced, validation failure returned
}
```

### Error Handling in Providers

**Multi-provider pattern (amazon-bedrock.ts):**
```typescript
try {
    // AWS SDK call
} catch (error) {
    if (error instanceof AwsError) {
        throw new Error(`AWS Bedrock error: ${error.message}`);
    }
    throw error;
}
```

### Error Propagation

- Errors propagate up via `throw` in stream wrapper (`stream.ts`)
- Async errors caught with `.catch()` at CLI entry point (`main().catch()`)
- Custom error types not evident - standard `Error` class used throughout

---

## 6. Logging Patterns

### Logging Strategy by Package

**1. mom (Slack bot) - Structured logging with chalk:**
```typescript
// src/log.ts - comprehensive structured logging
export function logUserMessage(ctx: LogContext, text: string): void {
    console.log(chalk.green(`${timestamp()} ${formatContext(ctx)} ${text}`));
}

export function logToolStart(ctx: LogContext, toolName: string, label: string, args: Record<string, unknown>): void {
    console.log(chalk.yellow(`${timestamp()} ${formatContext(ctx)} ↳ ${toolName}: ${label}`));
}
```

**2. coding-agent - Debug logging via env var:**
```typescript
// Uses PI_TUI_WRITE_LOG env var for debug output
// Configurable log directory path
export function getDebugLogPath(): string {
    return join(getAgentDir(), `${APP_NAME}-debug.log`);
}
```

**3. General - console.log usage (882 occurrences):**

| Pattern | Count |
|---------|-------|
| `console.log` | ~700 |
| `console.error` | ~100 |
| `console.warn` | ~50 |
| `console.debug` | ~30 |

### Logging Concerns

**Observations:**
- No structured logging library (no pino, winston, etc.)
- mom package has best practices with dedicated log module
- Other packages use ad-hoc console.log
- Debug logging only in mom package via `PI_TUI_WRITE_LOG`
- No log level configuration visible
- Secrets/keys appear to be filtered from logs (evidence: `<authenticated>` placeholder in env-api-keys.ts)

---

## 7. Config Management

### Configuration Hierarchy

**1. Package-level config (package.json piConfig):**
```typescript
const pkg = JSON.parse(readFileSync(getPackageJsonPath(), "utf-8"));
export const APP_NAME: string = pkg.piConfig?.name || "pi";
export const CONFIG_DIR_NAME: string = pkg.piConfig?.configDir || ".pi";
```

**2. Environment variables:**
```typescript
// Env var overrides with predictable naming
export const ENV_AGENT_DIR = `${APP_NAME.toUpperCase()}_CODING_AGENT_DIR`;
const baseUrl = process.env.PI_SHARE_VIEWER_URL || DEFAULT_SHARE_VIEWER_URL;
```

**3. User config files (JSON):**
```
~/.pi/agent/
    auth.json      - API keys and OAuth tokens
    settings.json  - User preferences
    models.json    - Custom model configurations
    sessions/      - Session history
```

### Settings Schema

**Type-safe settings via Typebox (coding-agent):**
```typescript
export interface Settings {
    lastChangelogVersion?: string;
    defaultProvider?: string;
    defaultModel?: string;
    defaultThinkingLevel?: "off" | "minimal" | "low" | "medium" | "high" | "xhigh";
    transport?: TransportSetting;
    compaction?: CompactionSettings;
    retry?: RetrySettings;
    // ... extensive configuration surface
}
```

---

## 8. Secrets Management

### API Key Resolution Strategy

**Pattern: Env var lookup with fallbacks (env-api-keys.ts):**
```typescript
export function getEnvApiKey(provider: KnownProvider): string | undefined {
    // Provider-specific env vars
    if (provider === "github-copilot") {
        return process.env.COPILOT_GITHUB_TOKEN || process.env.GH_TOKEN || process.env.GITHUB_TOKEN;
    }
    if (provider === "anthropic") {
        return process.env.ANTHROPIC_OAUTH_TOKEN || process.env.ANTHROPIC_API_KEY;
    }

    // Generic mapping
    const envMap: Record<string, string> = {
        openai: "OPENAI_API_KEY",
        google: "GEMINI_API_KEY",
        // ... more providers
    };
    return envVar ? process.env[envVar] : undefined;
}
```

### Secrets Storage

**auth.json structure:**
```typescript
// Stored encrypted or via OS keychain integration
// OAuth tokens stored via provider-specific OAuth flows
```

### Security Observations

- Secrets never logged (placeholders used: `"<authenticated>"`)
- No hardcoded credentials in source
- Env var pattern follows standard naming conventions
- OAuth token refresh handled automatically
- Vertex ADC credentials checked via `GOOGLE_APPLICATION_CREDENTIALS` or default path

---

## 9. package.json Script Structure

### Root Scripts

```json
{
    "scripts": {
        "clean": "npm run clean --workspaces",
        "build": "cd packages/tui && npm run build && cd ../ai && npm run build && ...",
        "dev": "concurrently --names \"ai,agent,coding-agent,mom,web-ui,tui\" ...",
        "check": "biome check --write --error-on-warnings . && tsgo --noEmit && npm run check:browser-smoke && cd packages/web-ui && npm run check",
        "test": "npm run test --workspaces --if-present",
        "prepublishOnly": "npm run clean && npm run build && npm run check",
        "publish": "npm run prepublishOnly && npm publish -ws --access public"
    }
}
```

### Per-Package Patterns

**Build command uniformity:**
- All packages use `tsgo -p tsconfig.build.json`
- Clean script uses `shx rm -rf dist`
- Dev uses `--watch --preserveWatchOutput`

**Test command variation:**
- vitest: `vitest --run` (ai, agent, coding-agent)
- Node test: `node --test --import tsx test/*.test.ts` (tui)

### Dependency Overrides

```json
"overrides": {
    "rimraf": "6.1.2",
    "fast-xml-parser": "5.3.8",
    "gaxios": { "rimraf": "6.1.2" }
}
```

---

## Recommendations

### Strengths to Maintain

1. **Comprehensive provider test coverage** - abort.test.ts pattern across providers
2. **Strict TypeScript** - strict mode enabled, declaration maps generated
3. **Consistent tooling** - Biome + TypeScript + vitest across all packages
4. **Conventional commits** - clear commit history with issue references
5. **Environment-based secrets** - no hardcoded credentials

### Areas for Improvement

1. **Logging standardization** - adopt structured logging (pino/winston) across packages
2. **Test coverage for mom/pods** - no visible tests for these packages
3. **Error type specificity** - consider custom error classes for better error handling
4. **Web UI testing** - no visible component tests for lit web components
5. **Browser E2E** - only smoke test, no Playwright/Cypress tests

### Code Review Gaps

1. **No PR review visible** - commits go directly to main
2. **Limited code owners** - unclear who reviews what
3. **No pre-commit hooks** - husky only runs `prepare` script, no lint checks on commit

---

## Quality Metrics Summary

| Dimension | Score | Notes |
|-----------|-------|-------|
| Test Coverage | 7/10 | Good unit/integration tests, missing mom/pods tests |
| Type Safety | 8/10 | Strict mode, declaration maps, selective any usage |
| Code Formatting | 9/10 | Biome consistent, 120 char line width |
| Error Handling | 7/10 | Try-catch patterns, but no custom error types |
| Logging | 5/10 | Ad-hoc console.log, mom has best pattern |
| Secrets Management | 8/10 | Env vars, no logging of secrets |
| CI/CD | 8/10 | Good coverage, cross-platform builds |
| Commit Quality | 8/10 | Conventional commits, issue references |
