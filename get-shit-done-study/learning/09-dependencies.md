# Dependency Analysis

## Philosophy: Minimal Dependencies

GSD follows a **zero-runtime-dependencies** philosophy for its core. This is a deliberate security and reliability choice.

## Runtime Dependencies

**None.** The `get-shit-done/bin/lib/*.cjs` modules use only Node.js built-ins.

## Dev Dependencies

```json
"devDependencies": {
  "c8": "^11.0.0",
  "esbuild": "^0.24.0"
}
```

| Package | Version | Purpose | Why Needed |
|---------|---------|---------|------------|
| `c8` | ^11.0.0 | Code coverage | Generates coverage reports for `npm run test:coverage` |
| `esbuild` | ^0.24.0 | Hook bundler | Bundles `hooks/*.js` into `hooks/dist/` for distribution |

## Dependency-free Core

### Modules Using Only Built-ins

Every module in `get-shit-done/bin/lib/` is dependency-free:

```
commands.cjs     — Command registration
config.cjs       — Configuration management
core.cjs         — Core workflow logic
frontmatter.cjs  — YAML frontmatter parsing
init.cjs         — Initialization
milestone.cjs    — Milestone management
model-profiles.c — Model configuration
phase.cjs        — Phase management
profile-output.cjs — Output formatting
profile-pipeline.cjs — Agent orchestration
roadmap.cjs      — Roadmap utilities
security.cjs     — Security utilities
state.cjs        — State management
template.cjs     — Template processing
uat.cjs          — User acceptance testing
verify.cjs       — Verification logic
workstream.cjs   — Workstream management
```

### Why No Dependencies?

1. **Security** — Every dependency is a potential attack vector
2. **Reliability** — No dependency conflicts, no breaking changes
3. **Performance** — No module resolution overhead
4. **Portability** — Works on any Node.js installation
5. **Trust** — Auditable code, no supply chain risk

## Hook Dependencies

The hooks in `hooks/` directory are compiled via esbuild:

```
hooks/
├── commands.cjs         # Hook command handlers
├── gsd-context-monitor.js
├── gsd-prompt-guard.js
├── gsd-workflow-guard.js
└── dist/                # esbuild output
    ├── gsd-context-monitor.js
    ├── gsd-prompt-guard.js
    └── gsd-workflow-guard.js
```

## Package Manager

- **npm** — Primary package manager
- **npm ci** — Used in CI for deterministic installs
- **package-lock.json** — Lockfile committed (38KB)

## Optional Dependencies

None declared. The project has no `optionalDependencies`.

## Peer Dependencies

None declared. The project has no `peerDependencies`.

## Transitive Dependencies

Because there are no runtime dependencies, the entire dependency tree for the published package is minimal:

```
get-shit-done-cc@1.29.0
└── (none — only bundled files)
```

## Why This Design?

### Comparison with Alternatives

| Approach | Pros | Cons |
|----------|------|------|
| **Zero deps (GSD)** | Secure, reliable, portable | More code to write/maintain |
| **Light deps** | Balance of features/safety | Still some attack surface |
| **Heavy deps** | Rich ecosystem | Maintenance burden, vulnerabilities |

### Security Implications

From `SECURITY.md`:
> Because GSD generates markdown files that become LLM system prompts, any user-controlled text flowing into planning artifacts is a potential indirect prompt injection vector.

By having zero runtime dependencies, GSD:
1. Eliminates supply chain attacks on npm
2. Prevents dependency-based exploits
3. Ensures reproducible behavior

## Dependency Updates

- **Dependabot:** Not explicitly configured (no GitHub Actions shown)
- **Manual updates:** Via `npm outdated` and `npm update`
- **Security patches:** Critical updates manually applied

## Build Dependencies

```bash
npm run build:hooks  # esbuild bundling
npm run prepublishOnly  # Runs build:hooks before publish
```

## Test Dependencies

Tests use Node.js built-ins only:
```javascript
const { describe, it, test, beforeEach, afterEach } = require('node:test');
const assert = require('node:assert/strict');
```

**Forbidden:** Jest, Mocha, Chai — no external testing frameworks allowed.

## Scripts Summary

| Script | Command | Dependencies |
|--------|---------|--------------|
| `build:hooks` | `node scripts/build-hooks.js` | esbuild |
| `prepublishOnly` | `npm run build:hooks` | esbuild |
| `test` | `node scripts/run-tests.cjs` | Node.js built-ins |
| `test:coverage` | `c8 ... node scripts/run-tests.cjs` | c8 |

## npm Package Contents

```json
"files": [
  "bin",
  "commands",
  "get-shit-done",
  "hooks/dist",
  "scripts"
]
```

Excludes:
- `tests/` — not needed by consumers
- `.github/` — CI config not needed in package
- Source hooks (`hooks/*.js`) — only bundled `dist/` included

## Dependency Health

| Metric | Status |
|--------|--------|
| Total runtime deps | 0 |
| Total dev deps | 2 |
| Known vulnerabilities | Minimal (2 dev deps, actively maintained) |
| License issues | None |

## What Could Be Dependencies (But Isn't)

| Feature | Alternative Used |
|---------|------------------|
| YAML parsing | Manual regex/string parsing in frontmatter.cjs |
| JSON validation | Node.js built-in JSON.parse with try/catch |
| File globbing | Node.js built-in path modules |
| HTTP requests | Node.js built-in fetch (Node 18+) |
| Date formatting | Node.js built-in Intl.DateTimeFormat |

## Conclusion

GSD's dependency philosophy is **pragmatic minimalism**:
- Zero runtime dependencies for core functionality
- Two dev dependencies for tooling (coverage, bundling)
- All features implemented with Node.js built-ins
- No external testing frameworks — Node's built-in test runner is sufficient

This approach prioritizes security, reliability, and long-term maintainability over developer convenience.
