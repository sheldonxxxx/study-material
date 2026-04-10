# Code Quality Practices

## Quality Assessment Summary

**Overall Posture:** Minimal tooling, maximal clarity -- trade-off driven by zero-dependency philosophy

## Testing Coverage

### Test Infrastructure

| Component | Test Coverage |
|-----------|---------------|
| WebSocket server (`server.cjs`) | Comprehensive (40+ unit tests, 25+ integration tests) |
| Plugin bootstrapping (`superpowers.js`) | None |
| Helper.js (browser client) | None |
| Skill markdown files | Integration tests only |
| Utility scripts (`render-graphs.js`) | None |

### Test Framework

The project uses Node.js native `assert` module only. No Mocha, Jest, or Vitest. Custom `test()` helper functions with pass/fail counting.

### WebSocket Protocol Testing Highlights

- Frame encoding/decoding (small/medium/large payloads)
- Boundary conditions (65535/65536 byte transitions)
- RFC 6455 compliance verification
- Multiple frames in single buffer
- Close frames with status codes
- Handshake key computation against RFC test vectors

### Integration Testing Highlights

- Server startup/shutdown lifecycle
- HTTP serving and file watching
- Concurrent clients and client cleanup
- Malformed JSON handling (graceful degradation)
- Platform-specific Windows lifecycle tests

## Type Safety

**Verdict: None.** Pure JavaScript project.

- No TypeScript
- No JSDoc with type checking
- No Flow
- No `checkJs` configuration

**Implications:**
- Runtime type errors only discovered through testing or execution
- Untyped Buffers, HTTP objects, and file system APIs in server.cjs
- All function parameters untyped

**Notable Pattern:** Manual frontmatter parser avoids YAML dependency:
```javascript
const extractAndStripFrontmatter = (content) => {
  const match = content.match(/^---\n([\s\S]*?)\n---\n([\s\S]*)$/);
  // ... manual key: value parsing
};
```

## Linting & Formatting

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
- Code style is implicit (2-space indentation in JS)
- PR review catches stylistic issues
- No pre-commit hooks

## Error Handling Patterns

### Server (`server.cjs`)

**JSON parsing errors - graceful degradation:**
```javascript
try {
  event = JSON.parse(text);
} catch (e) {
  console.error('Failed to parse WebSocket message:', e.message);
  return;  // Continues processing other messages
}
```

**Frame decoding errors - connection termination:**
```javascript
try {
  result = decodeFrame(buffer);
} catch (e) {
  socket.end(encodeFrame(OPCODES.CLOSE, Buffer.alloc(0)));
  clients.delete(socket);
  return;  // Closes connection on protocol violation
}
```

**Pattern:** Errors are logged to stderr and either cause graceful termination or are suppressed. No error propagation to callers.

### Helper.js

**Gap:** No try/catch around JSON.parse:
```javascript
ws.onmessage = (msg) => {
  const data = JSON.parse(msg.data);  // Can throw if malformed
  // ...
};
```

## Logging Practices

**Dual-purpose output:** Machine-readable JSON for parent process, human-readable for debugging.

```javascript
// IPC via JSON to stdout
console.log(JSON.stringify({ type: 'screen-added', file: filePath }));

// Errors to stderr
console.error('Failed to parse WebSocket message:', e.message);
```

**Observations:**
- No logger abstraction (no pino, winston, bunyan)
- No log levels (DEBUG, INFO, WARN, ERROR)
- No timestamps in output
- No structured metadata beyond event type

## Configuration Management

Environment variables only, no secrets (appropriate for plugin):

```javascript
const PORT = process.env.BRAINSTORM_PORT || (49152 + Math.floor(Math.random() * 16383));
const HOST = process.env.BRAINSTORM_HOST || '127.0.0.1';
const SESSION_DIR = process.env.BRAINSTORM_DIR || '/tmp/brainstorm';
```

## Code Review Practices

### PR Template Excellence

The `.github/PULL_REQUEST_TEMPLATE.md` is exceptionally thorough:

**Required sections:**
- Problem statement (specific failure mode)
- Change description (what, not why)
- Fitness for core library assessment
- Alternatives considered
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
- Human review is policy, not enforced by tooling

## Commit Message Style

No conventional commits. No commitlint. No enforced format.

Example patterns:
- `zero-dep brainstorm server RFC 6455`
- `fix windows owner pid detection`
- `add ws-protocol tests`

## Dependency Philosophy

**Minimal by design:**

| Location | Dependencies |
|----------|-------------|
| Root package.json | None (empty `dependencies`) |
| `tests/brainstorm-server/package.json` | `ws: ^8.19.0` (test only) |
| `render-graphs.js` | None (uses `execSync` with system `dot`) |

**Zero-dependency WebSocket implementation:** `server.cjs` implements RFC 6455 from scratch using only Node.js built-ins.

## Quality Summary

| Dimension | Rating | Notes |
|-----------|--------|-------|
| Test Coverage | Good (for what exists) | WebSocket server well-tested; plugin code lacks coverage |
| Type Safety | None | Pure JavaScript, no type checking |
| Linting/Formatting | None | No automated style enforcement |
| Error Handling | Adequate | Server handles errors gracefully; helper.js has gaps |
| Logging | Basic | JSON to stdout for IPC; no structured logging |
| Code Review | Rigorous template | Human-enforced, no automation |
| Dependencies | Minimal | Zero prod dependencies |

## Recommendations (Not Implemented)

1. Add ESLint + Prettier for JavaScript consistency
2. Add unit tests for `superpowers.js` plugin loading
3. Add error handling around `JSON.parse` in helper.js
4. Consider GitHub Actions for PR template checkbox enforcement
5. Consider TypeScript for server.cjs (best ROI: typed inputs/outputs)