OUTPUT_FILE: /Users/sheldon/Documents/claw/superpowers-study/research/07-code-quality.md

# Code Quality & Practices Assessment: Superpowers

## Overview

**Repository:** github.com/obra/superpowers (v5.0.6)
**Assessment scope:** JavaScript/CommonJS code, tests, tooling, and development practices
**Overall posture:** Minimal tooling, maximal clarity -- trade-off driven by zero-dependency philosophy

---

## 1. Testing Coverage

### Test Files

| File | Type | Purpose |
|------|------|---------|
| `tests/brainstorm-server/ws-protocol.test.js` | Unit | WebSocket frame encoding/decoding, RFC 6455 handshake |
| `tests/brainstorm-server/server.test.js` | Integration | Full brainstorm server (HTTP, WebSocket, file watching) |
| `tests/brainstorm-server/windows-lifecycle.test.sh` | Platform | Windows-specific lifecycle/shutdown behavior |

### Strengths

**WebSocket protocol has thorough unit tests** (`ws-protocol.test.js`):
- 40+ test cases covering frame encoding (small/medium/large), decoding, boundary conditions (65535/65536 bytes)
- RFC 6455 compliance verification (masked frames, unmasked server frames rejected)
- Handshake key computation verified against RFC test vectors
- Multiple frames in single buffer, close frames with status codes

**Integration tests are comprehensive** (`server.test.js`):
- 25+ test cases across categories: Server Startup, HTTP Serving, WebSocket Communication, File Watching
- Tests file watching debouncing, concurrent clients, client cleanup on close
- Tests malformed JSON handling (graceful degradation)
- Verifies helper.js API presence and frame template structure
- Includes negative test cases (404, non-HTML files ignored)

**Platform-specific tests** (`windows-lifecycle.test.sh`):
- Tests 6 scenarios including lifecycle check survival (75-second wait)
- Control test verifies bad OWNER_PID causes shutdown
- Clean shutdown verification

### Gaps

| Area | Coverage | Notes |
|------|----------|-------|
| Main plugin (`superpowers.js`) | **NONE** | No tests for plugin bootstrapping, skill loading, or config injection |
| Helper.js | **NONE** | No tests; `JSON.parse` in `ws.onmessage` can throw uncaught |
| Skill markdown files | **NONE** | Skills are content, not code; tested via integration tests in `tests/claude-code/` |
| `render-graphs.js` | **NONE** | Utility script has no automated tests |
| Error paths in server.cjs | **PARTIAL** | Tests verify graceful handling of malformed JSON, but missing coverage for file system errors (permissions, disk full) |

**Test framework:** Node.js native `assert` module only. No Mocha, Jest, or Vitest. Custom `test()` helper functions with pass/fail counting. Simple but functional.

---

## 2. Type Safety

**Verdict: None.** This is a pure JavaScript project.

- No TypeScript
- No JSDoc with type checking
- No Flow
- No `checkJs` in tsconfig (there is no tsconfig)

**Implications:**
- `server.cjs` uses untyped Buffers, HTTP objects, file system APIs
- `superpowers.js` uses `async`/`await` but all params are untyped
- `helper.js` uses implicit DOM types
- Runtime type errors only discovered through testing or execution

**Notable pattern:** The plugin uses a manual frontmatter parser rather than importing a YAML library:

```javascript
const extractAndStripFrontmatter = (content) => {
  const match = content.match(/^---\n([\s\S]*?)\n---\n([\s\S]*)$/);
  // ... manual key: value parsing
};
```

This avoids a dependency but shifts the type-safety burden to manual parsing.

---

## 3. Linting & Formatting

**Verdict: None configured.**

| Tool | Status |
|------|--------|
| ESLint | Not configured |
| Prettier | Not configured |
| .editorconfig | Not present |
| commitlint | Not configured |
| GitHub Actions CI | Not present |

**What this means:**
- No automated style enforcement
- Code style is implicit (2-space indentation in JS, standard bash in shell scripts)
- PR review must catch stylistic issues
- No pre-commit hooks

The project relies entirely on human code review for style consistency.

---

## 4. Error Handling Patterns

### Server (`server.cjs`)

**JSON parsing errors:**
```javascript
try {
  event = JSON.parse(text);
} catch (e) {
  console.error('Failed to parse WebSocket message:', e.message);
  return;  // Graceful degradation -- continues processing other messages
}
```

**Frame decoding errors:**
```javascript
try {
  result = decodeFrame(buffer);
} catch (e) {
  socket.end(encodeFrame(OPCODES.CLOSE, Buffer.alloc(0)));
  clients.delete(socket);
  return;  // Closes connection on protocol violation
}
```

**File watcher errors:**
```javascript
watcher.on('error', (err) => console.error('fs.watch error:', err.message));
// Watcher continues operating despite errors
```

**Pattern observation:** Errors are logged to stderr and either cause graceful termination or are suppressed. No error propagation to callers.

### Helper.js

**No try/catch around JSON.parse:**
```javascript
ws.onmessage = (msg) => {
  const data = JSON.parse(msg.data);  // Can throw if malformed
  if (data.type === 'reload') {
    window.location.reload();
  }
};
```

**Gap:** Malformed server messages will crash the helper script. This is arguably acceptable since helper.js runs in a browser context where uncaught exceptions are visible in the console.

### Systematic Debugging Skill Guidance

The `skills/systematic-debugging/root-cause-tracing.md` explicitly recommends:
> **In tests:** Use `console.error()` not logger - logger may be suppressed

This is followed consistently in test files.

---

## 5. Logging Practices

**Server IPC via JSON to stdout:**
```javascript
console.log(JSON.stringify({ type: 'screen-added', file: filePath }));
console.log(JSON.stringify({ source: 'user-event', ...event }));
```

**Startup/shutdown:**
```javascript
console.log(info);  // Human-readable server-started message
console.log(JSON.stringify({ type: 'server-stopped', reason }));
```

**Errors:**
```javascript
console.error('Failed to parse WebSocket message:', e.message);
console.error('fs.watch error:', err.message);
```

**Observations:**
- No logger abstraction (no pino, winston, bunyan)
- Output is dual-purpose: machine-readable JSON for parent process, human-readable for debugging
- No log levels (DEBUG, INFO, WARN, ERROR)
- No timestamps in output
- No structured metadata beyond event type

---

## 6. Configuration & Secrets Management

**Environment variable pattern:**
```javascript
const PORT = process.env.BRAINSTORM_PORT || (49152 + Math.floor(Math.random() * 16383));
const HOST = process.env.BRAINSTORM_HOST || '127.0.0.1';
const SESSION_DIR = process.env.BRAINSTORM_DIR || '/tmp/brainstorm';
```

**Path normalization with home directory expansion:**
```javascript
const normalizePath = (p, homeDir) => {
  if (normalized.startsWith('~/')) {
    normalized = path.join(homeDir, normalized.slice(2));
  }
  // ...
};
```

**No secrets management:** The project is a plugin/framework with no external service credentials. Plugin configuration is platform-native (hooks.json, plugin.json).

**No .env file handling:** Environment variables are set by the parent Claude Code process, not loaded from files.

---

## 7. Code Review Practices

### PR Template (`.github/PULL_REQUEST_TEMPLATE.md`)

The template is exceptionally thorough:

**Required sections:**
- Problem statement (specific failure mode, not "improving something")
- Change description (what, not why)
- Fitness for core library (is it general-purpose?)
- Alternatives considered
- No bundled unrelated changes
- Existing PR review (duplicates/prior art)
- Environment tested (harness, version, model)
- Evaluation methodology (before/after across multiple sessions)
- Rigor checklist (adversarial testing for skills content)
- Human review checkbox (mandatory)

**Notable requirements:**
- Skills changes require `superpowers:writing-skills` completion
- Behavior-shaping content requires eval evidence
- PRs closed without review if checkbox unchecked

### No Automated Enforcement

- No GitHub Actions CI
- No required status checks
- No branch protection rules enforced programmatically
- Human review is policy, not enforced by tooling

---

## 8. Commit Message Style

**No conventional commits.** No commitlint. No enforced format.

From project history (visible in release notes), commits are typically:
- Single-purpose
- Lowercase summary
- No rigid format (type: scope: subject)

Example patterns observed:
- `zero-dep brainstorm server RFC 6455`
- `fix windows owner pid detection`
- `add ws-protocol tests`

---

## 9. Dependency Philosophy

**Minimal by design:**

| Location | Dependencies |
|----------|-------------|
| Root package.json | None (empty `dependencies`) |
| `tests/brainstorm-server/package.json` | `ws: ^8.19.0` (test only) |
| `render-graphs.js` | None (uses `execSync` with system `dot`) |

**Zero-dependency WebSocket implementation:** `server.cjs` implements RFC 6455 from scratch using only Node.js built-ins (`crypto`, `http`, `fs`, `path`). The comment explicitly notes this:

> Uses the `ws` npm package as a test client (test-only dependency, not shipped to end users).

**Trade-off analysis:**
- Pro: No supply chain risk, no version conflicts, fast install
- Con: No type definitions, no battle-tested code, more code to maintain

---

## 10. Project Structure Quality

```
superpowers/
├── skills/              # Core content (markdown + scripts)
│   └── brainstorming/scripts/
│       ├── server.cjs        # Zero-dep WebSocket server
│       ├── helper.js         # Browser client script
│       └── frame-template.html
├── commands/            # Agent command definitions
├── agents/              # Agent prompt templates
├── hooks/               # Platform hooks
├── tests/
│   ├── brainstorm-server/
│   │   ├── server.test.js
│   │   ├── ws-protocol.test.js
│   │   └── windows-lifecycle.test.sh
│   └── claude-code/     # Integration tests via real sessions
├── docs/plans/          # Design documents
└── docs/superpowers/specs/  # Architectural specs
```

**Observations:**
- Clear separation: skills vs tests vs docs
- Platform-specific code isolated in `.claude-plugin/`, `.cursor-plugin/`, `.opencode/`, `.codex/`
- Test files co-located with tested code where reasonable
- No `src/` vs `lib/` distinction (flat structure appropriate for plugin)

---

## Summary Assessment

| Dimension | Rating | Notes |
|-----------|--------|-------|
| **Test Coverage** | Good (for what exists) | WebSocket server well-tested; plugin code and helper.js lack coverage |
| **Type Safety** | None | Pure JavaScript, no type checking |
| **Linting/Formatting** | None | No automated style enforcement |
| **Error Handling** | Adequate | Server handles errors gracefully; helper.js has gaps |
| **Logging** | Basic | JSON to stdout for IPC; no structured logging |
| **Config/Secrets** | Simple | Env vars only; no secrets (appropriate for plugin) |
| **Code Review** | Rigorous template | Human-enforced, no automation |
| **Dependencies** | Minimal | Zero prod dependencies |

### Key Strengths
1. **Zero-dependency WebSocket implementation** -- RFC-compliant, no supply chain risk
2. **Comprehensive WebSocket protocol tests** -- boundary conditions, RFC compliance
3. **Detailed PR template** -- enforces evaluation and adversarial testing for skills
4. **Multi-platform support** -- clean architectural separation

### Key Risks
1. **No type safety** -- runtime errors only caught in testing/execution
2. **No CI enforcement** -- PR template requirements unenforced
3. **No linting** -- style inconsistencies accumulate
4. **Plugin code untested** -- `superpowers.js` has no automated verification

### Recommendations (Not Implemented)
- Add ESLint + Prettier for JavaScript consistency
- Add unit tests for `superpowers.js` plugin loading
- Add error handling around `JSON.parse` in helper.js
- Consider GitHub Actions for PR template checkbox enforcement
- Consider TypeScript for server.cjs (best ROI: typed inputs/outputs, minimal new code)
