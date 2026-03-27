OUTPUT_FILE: /Users/sheldon/Documents/claw/superpowers-study/research/08-security-perf.md

# Security & Performance Patterns: Superpowers

## Project Overview

**Repository:** github.com/obra/superpowers
**Version:** 5.0.6
**Type:** Multi-platform AI coding agent plugin/framework
**License:** MIT

## Executive Summary

Superpowers is a skill-based framework for AI coding agents. Its security posture is strong for a local development tool due to zero-dependency architecture and localhost-only design. Performance patterns emphasize minimal resource usage through lazy loading and efficient file watching.

---

## Security Analysis

### 1. Zero-Dependency Architecture (Supply Chain Security)

**Finding:** The project eliminated 714 vendored files (express, ws, chokidar) by implementing a custom WebSocket server using only Node.js built-ins.

**Evidence:** `skills/brainstorming/scripts/server.cjs` lines 1-4:
```javascript
const crypto = require('crypto');
const http = require('http');
const fs = require('fs');
const path = require('path');
```

**Rationale:** From `docs/superpowers/specs/2026-03-11-zero-dep-brainstorm-server-design.md`:
> Vendoring node_modules into the git repo creates a supply chain risk: frozen dependencies don't get security patches, 714 files of third-party code are committed without audit, and modifications to vendored code look like normal commits.

**Security Benefit:** No external attack surface. All code is auditable in a single 355-line file.

---

### 2. WebSocket Protocol Compliance (RFC 6455)

**Finding:** The custom WebSocket implementation enforces strict RFC 6455 compliance.

**Evidence:** `server.cjs` line 48:
```javascript
if (!masked) throw new Error('Client frames must be masked');
```

**Additional protections:**
- Rejects unmasked client frames (required by RFC 6455 Section 5.1)
- Only handles TEXT (0x01), CLOSE (0x08), PING (0x09), PONG (0x0A) opcodes
- Unknown opcodes receive close frame with status 1003 (Unsupported Data)
- No binary frames or fragmented messages supported by design

**Evidence:** `server.cjs` lines 8, 196-216:
```javascript
const OPCODES = { TEXT: 0x01, CLOSE: 0x08, PING: 0x09, PONG: 0x0A };
// ...
default: {
  const closeBuf = Buffer.alloc(2);
  closeBuf.writeUInt16BE(1003);
  socket.end(encodeFrame(OPCODES.CLOSE, closeBuf));
  clients.delete(socket);
  return;
}
```

---

### 3. Path Traversal Prevention

**Finding:** Static file serving includes path traversal protection.

**Evidence:** `server.cjs` lines 145-156:
```javascript
} else if (req.method === 'GET' && req.url.startsWith('/files/')) {
  const fileName = req.url.slice(7);
  const filePath = path.join(CONTENT_DIR, path.basename(fileName));
  if (!fs.existsSync(filePath)) {
    res.writeHead(404);
    res.end('Not found');
    return;
  }
```

**Mechanism:** Using `path.basename()` strips directory components, preventing `../` traversal attacks.

---

### 4. Input Validation & Error Handling

**Finding:** WebSocket message handling includes JSON parsing error recovery.

**Evidence:** `server.cjs` lines 224-238:
```javascript
function handleMessage(text) {
  let event;
  try {
    event = JSON.parse(text);
  } catch (e) {
    console.error('Failed to parse WebSocket message:', e.message);
    return;
  }
  touchActivity();
  console.log(JSON.stringify({ source: 'user-event', ...event }));
```

**Behavior:** Malformed JSON is logged but does not crash the server. The test suite verifies this behavior: `tests/brainstorm-server/server.test.js` lines 284-296.

---

### 5. Defense-in-Depth Validation Pattern

**Finding:** The codebase includes documented defense-in-depth validation principles.

**Evidence:** `skills/systematic-debugging/defense-in-depth.md` documents four validation layers:

| Layer | Purpose | Example |
|-------|---------|---------|
| Entry Point | Reject invalid input at API boundary | `if (!workingDirectory) throw new Error()` |
| Business Logic | Ensure data makes sense for operation | Type and range checks |
| Environment Guards | Prevent dangerous operations in specific contexts | `NODE_ENV=test` restrictions |
| Debug Instrumentation | Capture context for forensics | Stack trace logging |

**Security Value:** Structured approach to preventing invalid data from causing bugs or security issues.

---

### 6. Secrets Management

**Finding:** No secrets, passwords, or authentication tokens in the codebase.

**Evidence:**
- No `.env` files in repository
- No hardcoded credentials in source
- Configuration via environment variables only (BRAINSTORM_PORT, BRAINSTORM_HOST, etc.)

**Context:** This is a localhost-only development server, so no authentication is required by design.

---

### 7. Hook System Security

**Finding:** Bash hook scripts use secure coding practices.

**Evidence:** `hooks/session-start` lines 1-6:
```bash
#!/usr/bin/env bash
# SessionStart hook for superpowers plugin

set -euo pipefail
```

**Additional security:**
- JSON string escaping via bash parameter substitution (lines 23-31) instead of external tools
- Platform-specific output handling to prevent double-injection

---

### Security Gaps & Recommendations

| Gap | Severity | Recommendation |
|-----|----------|----------------|
| No HTTPS/WSS support | Low | Localhost-only design makes this acceptable |
| No rate limiting on WebSocket | Low | localhost access only |
| No input sanitization for HTML content | Low | Content is written by AI agent, not user |
| Owner PID validation could be bypassed | Low | Documented limitation for WSL/Tailscale scenarios |

---

## Performance Analysis

### 1. Zero-Dependency Performance Benefits

**Finding:** Eliminating dependencies reduces startup time and memory footprint.

**Evidence:** `docs/superpowers/specs/2026-03-11-zero-dep-brainstorm-server-design.md`:
> A single `server.js` file (~250-300 lines) using `http`, `crypto`, `fs`, and `path`.

**Performance Impact:**
- No npm install required
- No node_modules directory (714 fewer files)
- Single file loads instantly

---

### 2. Efficient File Watching

**Finding:** The brainstorm server uses debounced fs.watch with intelligent change detection.

**Evidence:** `server.cjs` lines 256-298:
```javascript
const debounceTimers = new Map();

// In watcher callback:
if (debounceTimers.has(filename)) clearTimeout(debounceTimers.get(filename));
debounceTimers.set(filename, setTimeout(() => {
  debounceTimers.delete(filename);
  // Handle file change
}, 100));
```

**Optimization:** 100ms debounce prevents duplicate events (common on macOS `fs.watch`).

---

### 3. Buffer Accumulation Pattern

**Finding:** WebSocket frame decoding uses incremental buffer accumulation.

**Evidence:** `server.cjs` lines 179-194:
```javascript
let buffer = Buffer.alloc(0);
clients.add(socket);

socket.on('data', (chunk) => {
  buffer = Buffer.concat([buffer, chunk]);
  while (buffer.length > 0) {
    let result;
    try {
      result = decodeFrame(buffer);
    } catch (e) {
      socket.end(encodeFrame(OPCODES.CLOSE, Buffer.alloc(0)));
      clients.delete(socket);
      return;
    }
    if (!result) break;
    buffer = buffer.slice(result.bytesConsumed);
```

**Benefit:** Handles partial frames correctly by accumulating until complete.

---

### 4. Lazy Loading Design

**Finding:** Skills are loaded on-demand as markdown files, not eagerly initialized.

**Evidence:** `skills/brainstorming/visual-companion.md`:
> The server watches a directory for HTML files and serves the newest one to the browser.

**Performance:** No memory overhead for unused skills.

---

### 5. Server Lifecycle Management

**Finding:** Server automatically terminates to conserve resources.

**Evidence:** `server.cjs` lines 247-254, 319-324:
```javascript
const IDLE_TIMEOUT_MS = 30 * 60 * 1000; // 30 minutes
let lastActivity = Date.now();

function touchActivity() {
  lastActivity = Date.now();
}

// Lifecycle check every 60 seconds
const lifecycleCheck = setInterval(() => {
  if (!ownerAlive()) shutdown('owner process exited');
  else if (Date.now() - lastActivity > IDLE_TIMEOUT_MS) shutdown('idle timeout');
}, 60 * 1000);
```

**Benefit:** Prevents resource leaks from orphaned processes.

---

### 6. Port Allocation Strategy

**Finding:** Server uses random high port to avoid conflicts.

**Evidence:** `server.cjs` line 76:
```javascript
const PORT = process.env.BRAINSTORM_PORT || (49152 + Math.floor(Math.random() * 16383));
```

**Benefit:** Reduces port collision probability when multiple instances run.

---

### 7. WebSocket Broadcast Pattern

**Finding:** Simple broadcast to all connected clients.

**Evidence:** `server.cjs` lines 240-245:
```javascript
function broadcast(msg) {
  const frame = encodeFrame(OPCODES.TEXT, Buffer.from(JSON.stringify(msg)));
  for (const socket of clients) {
    try { socket.write(frame); } catch (e) { clients.delete(socket); }
  }
}
```

**Note:** O(n) iteration over all clients. For localhost development with few clients, this is acceptable. No evidence of scale concerns.

---

## Performance Patterns Summary

| Pattern | Implementation | Benefit |
|---------|---------------|---------|
| Zero dependencies | Custom WebSocket server | Fast startup, minimal memory |
| Debounced file watching | 100ms timeout per file | Reduces duplicate events |
| Buffer accumulation | Progressive Buffer.concat | Handles partial frames |
| Lazy loading | Skills as markdown files | No upfront cost |
| Auto-cleanup | 30-min idle timeout | Prevents resource leaks |
| Random port | High range selection | Reduces collision |

---

## Conclusion

### Security Posture

Superpowers demonstrates strong security awareness for a localhost development tool:

- **Strengths:** Zero-dependency architecture eliminates supply chain risk, RFC 6455 compliance ensures WebSocket security, path traversal protection prevents file system attacks, and defense-in-depth validation patterns are documented.
- **Trade-offs:** No encryption (acceptable for localhost), no authentication (by design), minimal input sanitization for HTML content (acceptable since content is AI-generated).

### Performance Posture

Performance is optimized for the use case:

- **Strengths:** Minimal resource usage through zero-dependency design, efficient file watching with debouncing, automatic cleanup of idle processes.
- **Trade-offs:** No caching layer for HTTP responses, simple O(n) WebSocket broadcast (acceptable for expected scale).

### Risk Assessment

| Category | Risk Level | Rationale |
|----------|------------|-----------|
| Supply Chain | Very Low | Zero external dependencies |
| Network Attack | Low | Localhost-only access |
| Data Exfiltration | Very Low | No sensitive data stored |
| Resource Exhaustion | Low | Auto-cleanup and idle timeout |
| Code Execution | Very Low | No eval, no user-provided code execution |

---

## References

- `skills/brainstorming/scripts/server.cjs` - Main server implementation
- `skills/systematic-debugging/defense-in-depth.md` - Validation patterns
- `docs/superpowers/specs/2026-03-11-zero-dep-brainstorm-server-design.md` - Architecture rationale
- `tests/brainstorm-server/server.test.js` - Integration tests
- `tests/brainstorm-server/ws-protocol.test.js` - Protocol unit tests
- `hooks/session-start` - Hook system implementation