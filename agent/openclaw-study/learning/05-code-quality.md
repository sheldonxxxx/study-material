# Code Quality - Learning Document

**Prerequisite:** Review `research/07-code-quality.md` for detailed analysis and examples.

---

## Overview

OpenClaw maintains **enterprise-grade code quality** with rigorous testing, strict typing, multi-layered linting, and comprehensive secrets detection. The project achieves an overall grade of **A (Excellent)** across all code quality dimensions.

---

## 1. Testing Infrastructure

### Framework: Vitest

- **Version:** Vitest 4.1.0 with V8 coverage provider
- **Execution Model:** Fork-based parallelization (`pool: "forks"`) — avoids instability from `threads`/`vmForks`
- **Test Location:** Colocated `*.test.ts` files alongside source code
- **Global Setup:** Comprehensive `test/setup.ts` with global mocks and state isolation

### Multiple Test Configurations

| Config | Purpose |
|--------|---------|
| `vitest.config.ts` | Main unit tests |
| `vitest.unit.config.ts` | Unit test subset |
| `vitest.gateway.config.ts` | Gateway-specific tests |
| `vitest.e2e.config.ts` | End-to-end tests |
| `vitest.live.config.ts` | Live/integration tests (require `OPENCLAW_LIVE_TEST=1`) |
| `vitest.channels.config.ts` | Channel tests |
| `vitest.extensions.config.ts` | Extension tests |
| `vitest.performance-config.ts` | Performance benchmarks |
| `ui/vitest.config.ts` | UI component tests |

### Coverage Thresholds

```yaml
lines: 70%
functions: 70%
branches: 55%
statements: 70%
```

Coverage is scoped to `./src/**/*.ts` only. Extensions and apps have separate testing.

### Test Isolation

```typescript
// From test/setup.ts
unstubEnvs: true,      // Automatic env restoration
unstubGlobals: true,    // Avoid cross-test pollution
```

- Environment variables automatically restored after each test
- Global state reset between tests
- Test home directory isolation via `withIsolatedTestHome()`
- Plugin registry state managed per-test

### Test Quality Indicators

- 100+ `.test.ts` files in `src/`
- 16 test fixture subdirectories in `test/fixtures/`
- 19 test helper modules in `test/helpers/`
- Architecture smell tests (`test/architecture-smells.test.ts`)
- Boundary tests for extension/plugin SDK (`test/extension-plugin-sdk-boundary.test.ts`)
- Live test pattern: `*.live.test.ts` suffix
- E2E test pattern: `*.e2e.test.ts` suffix

---

## 2. Type Safety

### Strict TypeScript Configuration

```json
// tsconfig.json
{
  "compilerOptions": {
    "strict": true,
    "noEmitOnError": true,
    "forceConsistentCasingInFileNames": true,
    "skipLibCheck": true,
    "module": "NodeNext",
    "moduleResolution": "NodeNext"
  }
}
```

### Key Settings

| Setting | Value | Implication |
|---------|-------|-------------|
| `strict` | `true` | Enables all strict type checks |
| `noEmitOnError` | `true` | Build fails if TypeScript reports errors |
| `forceConsistentCasingInFileNames` | `true` | Cross-platform consistency |
| `skipLibCheck` | `true` | Avoids d.ts conflicts with dependencies |

### No `any` Policy

```json
// .oxlintrc.json
{
  "rules": {
    "typescript/no-explicit-any": "error"
  }
}
```

**The `any` type is an error, not a warning.** The project explicitly forbids `any` via oxlint rule enforcement.

> "Never add `@ts-nocheck` and do not disable `no-explicit-any`; fix root causes and update Oxlint/Oxfmt config only when required." — CLAUDE.md

### ESM-Only

- `"type": "module"` in package.json
- No CommonJS throughout the codebase
- Path aliases for clean plugin SDK imports

---

## 3. Linting & Formatting

### TypeScript/JavaScript: Oxlint

```json
// .oxlintrc.json
{
  "plugins": ["unicorn", "typescript", "oxc"],
  "categories": {
    "correctness": "error",
    "perf": "error",
    "suspicious": "error"
  },
  "rules": {
    "typescript/no-explicit-any": "error",
    "curly": "error"
  },
  "ignorePatterns": [
    "assets/", "dist/", "extensions/", "node_modules/",
    "patches/", "pnpm-lock.yaml", "skills/", "vendor/"
  ]
}
```

### Swift: SwiftLint + SwiftFormat

```yaml
# .swiftlint.yml
analyzer_rules:
  - unused_declaration
  - unused_import

opt_in_rules:
  - array_init
  - closure_spacing
  - contains_over_first_not_nil
  - empty_count
  - fatal_error_message

disabled_rules:
  - trailing_whitespace  # SwiftFormat handles
  - todo  # Avoids forcing TODO resolution

force_cast: warning
force_try: warning

cyclomatic_complexity:
  warning: 20
  error: 120

function_body_length:
  warning: 150
  error: 300
```

### Shell: ShellCheck

```shell
# .shellcheckrc
disable=SC2034  # Variable appears unused
disable=SC2155  # Declare and assign separately
disable=SC2295  # Expansions inside ${..}
```

Only informational/warning-level issues are disabled — no critical issues.

### Formatting: Oxfmt

Uses default oxfmt configuration with `.oxfmtrc.jsonc`.

---

## 4. Error Handling Patterns

### Custom Error Type Pattern

```typescript
export class AcpRuntimeError extends Error {
  readonly code: AcpRuntimeErrorCode;
  override readonly cause?: unknown;

  constructor(code: AcpRuntimeErrorCode, message: string, options?: { cause?: unknown }) {
    super(message);
    this.name = "AcpRuntimeError";
    this.code = code;
    this.cause = options?.cause;
  }
}
```

### Error Code Registry

```typescript
export const ACP_ERROR_CODES = [
  "ACP_BACKEND_MISSING",
  "ACP_BACKEND_UNAVAILABLE",
  "ACP_BACKEND_UNSUPPORTED_CONTROL",
  "ACP_DISPATCH_DISABLED",
  "ACP_INVALID_RUNTIME_OPTION",
  "ACP_SESSION_INIT_FAILED",
  "ACP_TURN_FAILED",
] as const;
```

### Error Handling Utilities

- **Type guards:** `isAcpRuntimeError(value)`
- **Conversion:** `toAcpRuntimeError()`
- **Boundary helpers:** `withAcpRuntimeErrorBoundary()`
- **Cause chaining:** Uses Error cause API for debugging

---

## 5. Logging Patterns

### Logger Architecture

```typescript
// From src/logging/console.ts
export type ConsoleStyle = "pretty" | "compact" | "json";

type ConsoleSettings = {
  level: LogLevel;
  style: ConsoleStyle;
};
```

### Output Styles

| Style | Use Case |
|-------|----------|
| `pretty` | TTY terminals — human readable |
| `compact` | Piped output — less verbose |
| `json` | Automation — structured logs |

### Key Features

- Console output routed to stderr (stdout clean for RPC/JSON modes)
- Console capture enabled for file logging
- Subsystem-level filtering
- Environment variable log level override
- EPIPE error handling for broken pipes
- Tests default to silent logging (unless `OPENCLAW_TEST_CONSOLE=1`)

```typescript
if (process.env.VITEST === "true" && process.env.OPENCLAW_TEST_CONSOLE !== "1") {
  return "silent";  // Tests default to silent logging
}
```

---

## 6. Configuration Management

### Config Loading Features

From `src/config/io.ts`:

- JSON5 support for config files
- Environment variable substitution (`${VAR}`)
- Include directives (`$include`)
- Schema validation
- Version tracking (`meta.lastTouchedVersion`)
- Health monitoring with fingerprinting
- Audit logging

### Config Health Monitoring

```typescript
type ConfigHealthFingerprint = {
  hash: string;
  bytes: number;
  mtimeMs: number | null;
  ctimeMs: number | null;
  hasMeta: boolean;
  gatewayMode: string | null;
  observedAt: string;
};
```

**Anomaly Detection:**
- Config size drop detection (flags when config shrinks >50%)
- Missing meta detection
- Gateway mode removal detection
- Automatic clobbered config snapshots

### Config Audit Trail

Every config write is logged with full process context:

```typescript
type ConfigWriteAuditRecord = {
  ts: string;
  source: "config-io";
  event: "config.write";
  result: ConfigWriteAuditResult;
  configPath: string;
  pid: number;
  ppid: number;
  cwd: string;
};
```

### Permission Hardening

```typescript
// State dir: 0o700 (owner only)
await params.fsModule.promises.chmod(configDir, 0o700);
```

- Automatic permission tightening for state directories
- 0o600 for config files
- 0o700 for directories
- Graceful fallback with missing env vars (degraded mode, not crash)

---

## 7. Secrets Detection

### detect-secrets Integration

**Active Detectors (26 types):**
- AWS, Azure, GitHub, GitLab, Discord, Slack tokens
- OpenAI, Stripe, Twilio, Telegram keys
- Private keys, JWT tokens
- High-entropy string detection (Base64, Hex)
- Custom keyword detector

### Baseline

`.secrets.baseline` is a large file (~433KB) with extensive allowlist entries for known false positives.

### Exclusion Patterns

From `.detect-secrets.cfg`:
- `pnpm-lock.yaml` (package integrity blobs)
- UI label strings referencing API keys
- Schema labels for password fields
- Sparkle appcast signatures

### CI Integration

- Runs in pre-commit hooks
- Runs in CI pipeline
- Blocks commits with detected secrets

---

## 8. Code Quality Summary

| Category | Grade | Key Practices |
|----------|-------|--------------|
| Testing Infrastructure | A | Vitest, fork pool, 70% coverage, isolation |
| Type Safety | A | Strict mode, no `any`, noEmitOnError |
| Linting | A | Oxlint, SwiftLint, ShellCheck, ruff |
| Code Review | A | Comprehensive PR template, security required |
| Error Handling | A- | Custom errors, typed codes, cause chaining |
| Logging | A | Centralized, multi-mode, file capture |
| Config Management | A+ | Enterprise-grade, audit, health, validation |
| Secrets Detection | A | 26 detectors, pre-commit + CI |
| CI/CD | A | Multi-platform, multiple release channels |

---

## Key Takeaways

1. **Strict typing is non-negotiable** — `any` is an error, builds fail on type errors
2. **Test isolation is paramount** — Fork pool, env/globals reset, per-test state
3. **Multi-language linting** — TS, Swift, Shell, Python all have dedicated linters
4. **Secrets detection is automated** — Pre-commit + CI prevents accidental leaks
5. **Config management is enterprise-grade** — Health monitoring, audit trails, anomaly detection
6. **Comprehensive PR template** — Forces security review, root cause analysis, human verification

---

## References

- Detailed analysis: `research/07-code-quality.md`
- Test configuration: `vitest.config.ts`, `vitest.unit.config.ts`
- Linting: `.oxlintrc.json`, `.swiftlint.yml`, `.shellcheckrc`
- Secrets: `.secrets.baseline`, `.detect-secrets.cfg`
- Logging: `src/logging/console.ts`
- Config: `src/config/io.ts`
