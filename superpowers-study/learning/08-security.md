# Security Patterns

## Security Posture Summary

Superpowers demonstrates strong security awareness for a localhost development tool. Zero-dependency architecture eliminates supply chain risk, and RFC 6455 compliance ensures WebSocket security.

**Risk Assessment:**

| Category | Risk Level | Rationale |
|----------|------------|-----------|
| Supply Chain | Very Low | Zero external dependencies |
| Network Attack | Low | Localhost-only access |
| Data Exfiltration | Very Low | No sensitive data stored |
| Resource Exhaustion | Low | Auto-cleanup and idle timeout |
| Code Execution | Very Low | No eval, no user-provided code execution |

---

## Zero-Dependency Architecture (Supply Chain Security)

**Finding:** The project eliminated 714 vendored files by implementing a custom WebSocket server using only Node.js built-ins.

**Implementation (`server.cjs` lines 1-4):**
```javascript
const crypto = require('crypto');
const http = require('http');
const fs = require('fs');
const path = require('path');
```

**Security Benefit:** No external attack surface. All code is auditable in a single 355-line file.

**Rationale:** Vendoring node_modules creates supply chain risk: frozen dependencies do not get security patches, 714 files of third-party code are committed without audit, and modifications to vendored code look like normal commits.

---

## WebSocket Protocol Compliance (RFC 6455)

**Finding:** The custom WebSocket implementation enforces strict RFC 6455 compliance.

**Mask enforcement (`server.cjs` line 48):**
```javascript
if (!masked) throw new Error('Client frames must be masked');
```

**Additional protections:**
- Rejects unmasked client frames (required by RFC 6455 Section 5.1)
- Only handles TEXT (0x01), CLOSE (0x08), PING (0x09), PONG (0x0A) opcodes
- Unknown opcodes receive close frame with status 1003 (Unsupported Data)
- No binary frames or fragmented messages supported by design

---

## Path Traversal Prevention

**Finding:** Static file serving includes path traversal protection.

**Implementation (`server.cjs` lines 75-82):**
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

## Input Validation & Error Handling

**JSON parsing errors - graceful degradation:**
```javascript
function handleMessage(text) {
  let event;
  try {
    event = JSON.parse(text);
  } catch (e) {
    console.error('Failed to parse WebSocket message:', e.message);
    return;
  }
  // ... continue processing
}
```

**Behavior:** Malformed JSON is logged but does not crash the server.

---

## Defense-in-Depth Validation

**Finding:** The codebase includes documented defense-in-depth validation principles.

**Four validation layers:**

| Layer | Purpose | Example |
|-------|---------|---------|
| Entry Point | Reject invalid input at API boundary | `if (!workingDirectory) throw new Error()` |
| Business Logic | Ensure data makes sense for operation | Type and range checks |
| Environment Guards | Prevent dangerous operations in specific contexts | `NODE_ENV=test` restrictions |
| Debug Instrumentation | Capture context for forensics | Stack trace logging |

---

## Secrets Management

**Finding:** No secrets, passwords, or authentication tokens in the codebase.

**Evidence:**
- No `.env` files in repository
- No hardcoded credentials in source
- Configuration via environment variables only (BRAINSTORM_PORT, BRAINSTORM_HOST, etc.)

**Context:** This is a localhost-only development server, so no authentication is required by design.

---

## Hook System Security

**Bash hook scripts use secure coding practices:**

```bash
#!/usr/bin/env bash
set -euo pipefail
```

**Additional security:**
- JSON string escaping via bash parameter substitution instead of external tools
- Platform-specific output handling to prevent double-injection

---

## Security Gaps & Risk Acceptances

| Gap | Severity | Acceptance Rationale |
|-----|----------|----------------------|
| No HTTPS/WSS support | Low | Localhost-only design |
| No rate limiting on WebSocket | Low | localhost access only |
| No input sanitization for HTML content | Low | Content is written by AI agent, not user |
| Owner PID validation could be bypassed | Low | Documented limitation for WSL/Tailscale scenarios |

---

## Performance Patterns

### Zero-Dependency Performance Benefits

- No npm install required
- No node_modules directory (714 fewer files)
- Single file loads instantly

### Efficient File Watching

**Debounced fs.watch (`server.cjs` lines 256-298):**
```javascript
const debounceTimers = new Map();

if (debounceTimers.has(filename)) clearTimeout(debounceTimers.get(filename));
debounceTimers.set(filename, setTimeout(() => {
  debounceTimers.delete(filename);
  // Handle file change
}, 100));
```

**Benefit:** 100ms debounce prevents duplicate events common on macOS `fs.watch`.

### Buffer Accumulation Pattern

**Progressive buffer handling (`server.cjs` lines 215-227):**
```javascript
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

### Auto-Cleanup Lifecycle

**Idle timeout (`server.cjs` lines 251-262):**
```javascript
const IDLE_TIMEOUT_MS = 30 * 60 * 1000; // 30 minutes
let lastActivity = Date.now();

function touchActivity() {
  lastActivity = Date.now();
}

const lifecycleCheck = setInterval(() => {
  if (!ownerAlive()) shutdown('owner process exited');
  else if (Date.now() - lastActivity > IDLE_TIMEOUT_MS) shutdown('idle timeout');
}, 60 * 1000);
```

**Benefit:** Prevents resource leaks from orphaned processes.

### Random Port Allocation

```javascript
const PORT = process.env.BRAINSTORM_PORT || (49152 + Math.floor(Math.random() * 16383));
```

**Benefit:** Reduces port collision probability when multiple instances run.

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

**Strengths:**
- Zero-dependency architecture eliminates supply chain risk
- RFC 6455 compliance ensures WebSocket security
- Path traversal protection prevents file system attacks
- Defense-in-depth validation patterns are documented

**Trade-offs:**
- No encryption (acceptable for localhost)
- No authentication (by design)
- Minimal input sanitization for HTML content (acceptable since content is AI-generated)

### Performance Posture

**Strengths:**
- Minimal resource usage through zero-dependency design
- Efficient file watching with debouncing
- Automatic cleanup of idle processes

**Trade-offs:**
- No caching layer for HTTP responses
- Simple O(n) WebSocket broadcast (acceptable for expected localhost scale)