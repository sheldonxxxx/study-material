# OpenClaw Feature Batch Report: Features 10-12

**Project:** OpenClaw - Personal AI Assistant
**Report:** Batch 4 - Security & Sandboxing, Media Pipeline, Automation
**Date:** 2026-03-26
**Source:** Feature index, topology, and deep code exploration

---

## Feature 10: Security & Sandboxing

### Overview
Comprehensive security model with secure defaults, pairing-based access control, per-session Docker sandboxing, and fine-grained tool permissions. The security system spans 40 files in `src/security/` and 55 files in `src/secrets/`.

### Key Components

#### 1. Audit System (`src/security/audit.ts`, `audit.test.ts`)
The `SecurityAuditReport` provides deep security scanning:
- **Filesystem checks**: Permission auditing, Windows ACL validation
- **Channel security**: Per-channel allowlist enforcement
- **Dangerous config detection**: Flags insecure flag combinations
- **Gateway probing**: Optional deep probe to verify gateway connectivity
- **Tool policy enforcement**: Sandbox tool allowlists/denylists

**Key design**: Lazy-loaded audit modules via dynamic imports (`audit.nondeep.runtime.ts`, `audit.deep.runtime.ts`) to keep cold startup fast.

#### 2. DM Policy & Access Control (`src/security/dm-policy-shared.ts`)
Resolves DM/group access decisions with layered policy sources:

```
resolveDmGroupAccessDecision:
  - dmPolicy: "open" | "pairing" | "allowlist" | "disabled"
  - groupPolicy: "open" | "allowlist" | "disabled"
  - Effective allowFrom merged from config + pairing store
  - Returns: { decision: "allow" | "block" | "pairing", reasonCode, reason }
```

**Key insight**: The pairing store is DM-only; group auth explicitly does NOT inherit DM pairing approvals (line 262: "Group command authorization must not inherit DM pairing-store approvals").

#### 3. Docker Sandbox (`src/agents/sandbox/`)
Per-session Docker isolation with pluggable backends:

**Backend types**:
- `docker-backend.ts`: Docker containers (default for non-main sessions)
- `ssh-backend.ts`: SSH-based remote sandbox
- `browser.ts`: Browser automation sandbox with VNC/noVNC

**Sandbox config resolution**:
```typescript
resolveSandboxConfigForAgent:
  - scope: "agent" | "session" | "shared"
  - agent-specific overrides merged with global config
  - env vars: LANG, custom env merges
  - binds: workspace mounts from host
```

**Docker execution** (`docker.ts`):
- Uses `spawn()` with piped stdio (not shell by default)
- Windows spawn resolution via `windows-spawn.js`
- Abort signal support for cancellation
- Error handling: ENOENT -> friendly "Docker not found" message

**Security validation** (`validate-sandbox-security.ts`):
- Validates container targets don't escape intended scope
- Checks dangerous boolean flags like `dangerouslyAllowReservedContainerTargets`
- FS bridge path safety enforcement

#### 4. Tool Policy (`src/agents/sandbox/tool-policy.ts`)
Per-agent sandbox tool allowlists:
```typescript
resolveSandboxToolPolicyForAgent:
  - allow: string[] (empty = allow all)
  - deny: string[] (denied tools)
  - alsoAllow: string[] (extra permissions)
  - Priority: agent config > global config > defaults
```

**Defaults**:
- `DEFAULT_TOOL_ALLOW`: Base allowlist
- `DEFAULT_TOOL_DENY`: Base denylist

#### 5. Secrets Management (`src/secrets/`)
Provider-based secrets with multiple backends:

**Provider types**:
- `env`: Environment variables
- `file`: JSON/env files (`~/.openclaw/secrets/`)
- `exec`: External secret executables

**Secret resolution** (`resolve.ts`):
- Recursive secret ref resolution with depth limit
- Auth profile propagation
- Provider env var injection

**Configuration** (`configure.ts`):
- Interactive secrets setup wizard
- CSV parsing for allowlists
- Windows path detection (UNC, drive letters)

#### 6. External Content Security (`src/security/external-content.ts`)
Wraps untrusted external content with security markers:

```typescript
// Unique boundary markers per wrapper
createExternalContentMarkerId() -> randomBytes(8).toString("hex")

// Suspicious pattern detection (prompt injection)
SUSPICIOUS_PATTERNS:
  - ignore previous instructions
  - forget everything
  - new instructions:
  - system prompt override
  - exec/rm -rf commands
  - <system> tags
```

**Content source types**: email, webhook, api, browser, channel_metadata, web_search, web_fetch

#### 7. Dangerous Tools Registry (`src/security/dangerous-tools.ts`)
Centralized tool risk constants:

```typescript
// Gateway HTTP POST /tools/invoke denylist
DEFAULT_GATEWAY_HTTP_TOOL_DENY:
  - sessions_spawn (RCE risk)
  - sessions_send (cross-session injection)
  - cron (persistent automation)
  - gateway (control plane)
  - whatsapp_login (interactive flow)

// ACP tools requiring explicit approval
DANGEROUS_ACP_TOOL_NAMES:
  - exec, spawn, shell, sessions_spawn, sessions_send
  - gateway, fs_write, fs_delete, fs_move, apply_patch
```

#### 8. Security Doctor Fix (`src/security/fix.ts`)
Automated remediation:
- **chmod**: Sets correct permissions (0o700 for dirs, 0o600 for files)
- **icacls**: Windows ACL reset commands
- **Symlink handling**: Skips symlinks safely
- **ENOENT handling**: Gracefully skips missing paths

#### 9. Skill Scanning (`src/security/skill-scanner.ts`)
Scans skill packages for security issues:
- Pattern: Looks for dangerous API usage, insecure patterns
- Caching: File scan cache (5000 entries), dir entry cache
- Max files: 500 default, 1MB per file max
- Scannable extensions: .js, .ts, .mjs, .cjs, .jsx, .tsx

### Notable Patterns & Shortcuts

1. **Lazy module loading**: Security audit modules loaded on-demand via dynamic imports to avoid cold startup bloat
2. **Centralized danger constants**: One source of truth for tool risks shared across gateway HTTP restrictions, security audits, and ACP prompts
3. **Failover-safe errors**: Secrets and model failures use typed failover errors that bubble up properly
4. **Windows-first spawn handling**: Docker spawn uses `windows-spawn.js` for proper Windows support

### Technical Debt / Concerns

1. **Sandbox Docker dependency**: Requires Docker in PATH; friendly error message exists but Docker must be pre-installed
2. **Complex allowlist merging**: Tool policy merging logic is intricate with multiple precedence rules
3. **Test file sizes**: `audit.test.ts` is 125KB+ - very large test file suggests complex test scenarios

---

## Feature 11: Media Pipeline

### Overview
Unified media processing pipeline for images, audio, and video with transcription hooks, size caps, and temp file lifecycle management. Spans `src/media/` (50 files), `src/media-understanding/` (58 files), and `src/image-generation/` (11 files).

### Key Components

#### 1. Media Store (`src/media/store.ts`)
Central media file management:

```typescript
MEDIA_MAX_BYTES = 5 * 1024 * 1024  // 5MB default
MEDIA_FILE_MODE = 0o644  // Intentional readable for Docker sandbox containers
DEFAULT_TTL_MS = 2 * 60 * 1000  // 2 minutes temp file TTL
```

**Key functions**:
- `ensureMediaDir()`: Creates `~/.openclaw/media/` with 0o700 permissions
- `extractOriginalFilename()`: Extracts from pattern `{original}---{uuid}.{ext}`
- `sanitizeFilename()`: Cross-platform safe, keeps alphanumeric/dots/hyphens/underscores/Unicode

**Network fetch** (`fetch.ts`):
- HTTP/HTTPS support
- SSRF protection via `resolvePinnedHostname`
- Response size limits
- Retry logic with directory recreation on ENOENT

#### 2. Image Processing (`src/media/image-ops.ts`)
Uses `sharp` library (with `sips` fallback on macOS/Bun):

**Capabilities**:
- EXIF orientation reading (handles JPEG rotation)
- HEIC->JPEG conversion
- Image resizing with quality reduction steps: [85, 75, 65, 55, 45, 35]
- Alpha channel detection
- PNG optimization

**Resize grid**: [1800, 1600, 1400, 1200, 1000, 800]px (max side)

**Backend selection**:
```typescript
prefersSips():
  - env OPENCLAW_IMAGE_BACKEND=sips
  - or (Bun + macOS + no explicit backend)
```

#### 3. Web Media (`src/media/web-media.ts`)
Handles remote media fetching with optimization:

```typescript
WebMediaOptions:
  - maxBytes: size cap
  - optimizeImages: auto-compress
  - ssrfPolicy: SSRF protection
  - localRoots: allowed read paths
  - sandboxValidated: caller-guarded reads
```

**HEIC handling**: Detected via MIME type or file extension, converted to JPEG

#### 4. Media Understanding Pipeline (`src/media-understanding/apply.ts`)
Applies media understanding to message contexts:

**Capability order**: image -> audio -> video (processed in sequence)

**Supported MIME types**:
- Images: All standard image types
- Audio: audio/* plus text transcripts
- Video: video/* plus frame extraction
- Files: text/* types (CSV, TSV, Markdown, logs, config, JSON, YAML, XML)

**Transcript handling**:
- Audio: Extracted via transcription runner
- Video: Frame extraction + transcription
- Format: Human-readable transcript blocks

#### 5. Media Understanding Runner (`src/media-understanding/runner.ts`)
Manages media understanding model invocation:

**Provider registry**: Map of provider ID to MediaUnderstandingProvider

**Binary detection**:
```typescript
candidateBinaryNames():
  - Non-Windows: [name]
  - Windows: [name] + [name + each PATHEXT extension]
```

**Path resolution**: Expands ~ to $HOME, checks path separators

**Model decision**:
```typescript
buildModelDecision():
  - Primary model from config
  - Fallback chain
  - Provider selection
```

#### 6. Audio Transcription (`src/media-understanding/runner.ts`)
Multiple transcription backends:

**Auto-detection**: Provider preference order
**Deepgram**: Live and standard audio transcription
**OpenAI Whisper**: Via `extensions/openai-whisper/`
**Provider registry**: Extensible via `buildMediaUnderstandingRegistry()`

#### 7. Image Generation (`src/image-generation/runtime.ts`)
Model-agnostic image generation with fallback:

```typescript
generateImage():
  - Candidates: primary + fallback models
  - Per-provider: generateImage() call
  - Error handling: FailoverError with reason/status/code
  - Result: { images, provider, model, attempts, metadata }
```

**No-config message**: Helpful error if no image generation model configured

**Fallback reporting**: Shows all failed attempts with error details

#### 8. Media Constants (`src/media/constants.ts`)
```typescript
MEDIA_KINDS: "image" | "audio" | "video" | "file"
MAX_BYTES_BY_KIND: size caps per media type
```

#### 9. MIME Detection (`src/media/mime.ts`)
- Sniffing from base64 strings
- Extension->MIME mapping
- MIME->extension mapping

#### 10. Inbound Path Policy (`src/media/inbound-path-policy.ts`)
Controls where media can be read from:
- Default local roots: `~/.openclaw/media/`, temp dirs
- iMessage attachment roots (macOS-specific)
- Merges configured + default roots

### Notable Patterns & Shortcuts

1. **Temp file lifecycle**: Short 2-minute TTL for temp media files
2. **Docker-readable permissions**: Media files use 0o644 so Docker containers can access mounted media
3. **HEIC as first-class citizen**: Native HEIC detection and conversion
4. **Capability-based routing**: Media understanding decides which models to use based on content type
5. **Sharp with Bun/sips fallback**: Graceful degradation when sharp isn't ideal

### Technical Debt / Concerns

1. **Provider env var coupling**: Image generation hard-codes env var patterns per provider
2. **Complex MIME sniffing**: Multiple detection strategies (extension, content-type, base64 sniff)
3. **Limited video support**: Video understanding relies on frame extraction + transcription, not native video models

---

## Feature 12: Automation (Cron, Webhooks, MCP)

### Overview
Automation capabilities including scheduled cron jobs, webhook triggers, and MCP (Model Context Protocol) integration. Spans `src/cron/` (79 files) with service/or jobs/timer/store subsystems.

### Key Components

#### 1. Cron Service (`src/cron/service.ts`)
Main orchestrator for scheduled automation:

```typescript
class CronService:
  - start() / stop()
  - list() / listPage()
  - add() / update() / remove()
  - run() / enqueueRun()  // Manual trigger
  - wake()  // Session wakeup
  - getJob()
```

#### 2. Cron Operations (`src/cron/service/ops.ts`)
CRUD and lifecycle management:

**Job lifecycle**:
```
createJob -> add() -> persist()
update() -> recomputeNextRunAtMs -> persist()
remove() -> filter from store
run() -> executeJobCoreWithTimeout -> update state
```

**Startup behavior**:
- Loads persisted jobs
- Arms timers for due jobs
- Catches up missed jobs on restart
- Run-log pruning for completed runs

#### 3. Job State Machine (`src/cron/service/jobs.ts`)
Complex job state management:

```typescript
CronJob:
  - schedule: at | every | cron
  - sessionTarget: main | isolated | current | session:N
  - payload: systemEvent | agentTurn
  - delivery: none | announce | webhook
  - failureAlert: optional alert config
  - state: nextRunAtMs, runningAtMs, lastRunAtMs, lastStatus...
```

**Schedule types**:
```typescript
{ kind: "at", at: string }           // One-time
{ kind: "every", everyMs: number }     // Recurring interval
{ kind: "cron", expr: string, tz?: string }  // Cron expression
```

**Stagger support**: Optional staggerMs for cron jobs to spread load

#### 4. Timer System (`src/cron/service/timer.ts`)
Time-based job triggering (41KB, most complex module):

```typescript
// Timer arming
armTimer() -> setTimeout with computed delay
stopTimer() -> clearTimeout

// Job execution
executeJobCoreWithTimeout():
  - timeout enforcement
  - heartbeat monitoring
  - delivery state tracking
  - telemetry collection
```

**Heartbeat policy**:
- Configurable interval
- Suppressed after timeout
- Tracks job liveness

**Wake modes**:
- `now`: Immediate execution
- `next-heartbeat`: Wait for next heartbeat

#### 5. Cron Schedule Parsing (`src/cron/schedule.ts`)
Uses `croner` library for cron evaluation:

```typescript
CRON_EVAL_CACHE_MAX = 512  // LRU cache of compiled cron objects

resolveCronTimezone():
  - From schedule config
  - Fallback: Intl.DateTimeFormat().resolvedOptions().timeZone

computeNextRunAtMs():
  - "at": Parse absolute time
  - "every": Calculate from anchor + elapsed
  - "cron": croner.nextRun()
```

#### 6. Store/Persistence (`src/cron/service/store.ts`)
JSON-based job persistence:

```typescript
CronStoreFile:
  version: 1
  jobs: CronJob[]

ensureLoaded()  // Load or initialize
persist()       // Write atomically
```

**Migration support**: Store migration between versions

#### 7. Delivery System (`src/cron/delivery.ts`)
Result delivery after job execution:

```typescript
CronDelivery:
  mode: "none" | "announce" | "webhook"
  channel?: ChannelId | "last"
  to?: string
  accountId?: string
  bestEffort?: boolean
  failureDestination?: CronFailureDestination
```

**Modes**:
- `announce`: Via messaging channel
- `webhook`: HTTP POST to configured URL
- `none`: Fire and forget

#### 8. Webhook URL Validation (`src/cron/webhook-url.ts`)
```typescript
normalizeHttpWebhookUrl():
  - Must be http: or https:
  - Trim whitespace
  - URL parse validation
  - Returns null for invalid
```

#### 9. Gateway HTTP Server (`src/gateway/server-http.ts`)
Handles incoming webhook HTTP requests:

**Hook dispatch**:
```typescript
handleHookHttpRequest():
  - Token extraction
  - Body parsing (JSON)
  - Agent routing via hook config
  - Rate limiting (20 failures per 60s window)
  - External content source detection (gmail vs webhook)
```

**Hook auth**: HMAC-based token verification

**Webhook handling**:
- `/hooks/gmail`: Gmail Pub/Sub integration
- `/hooks/webhook`: Generic webhook trigger
- `/hooks/{agentId}`: Agent-specific hooks

#### 10. MCP Integration (`skills/mcporter/SKILL.md`)
External MCP bridge via mcporter CLI:

**Capabilities**:
- List MCP servers and tools
- Call tools directly
- OAuth authentication flow
- Config management
- Daemon mode for persistent servers

**CLI interface**:
```bash
mcporter list [--schema]
mcporter call <server.tool> key=value
mcporter auth <server> [--reset]
mcporter daemon start|status|stop|restart
mcporter generate-cli --server <name>
```

**Config**: `./config/mcporter.json` (configurable)

### Notable Patterns & Shortcuts

1. **croner library**: Leverages `croner` for cron expression parsing with timezone support and caching
2. **Heartbeat-based wake**: Cron jobs can "wake" sessions via heartbeat mechanism, not just immediate execution
3. **Delivery awareness**: Jobs track delivery status separately from execution status
4. **Stagger for cron**: Built-in staggerMs to spread cron job execution across time window
5. **Gmail Pub/Sub**: Native Gmail webhook integration via hook subsystem
6. **Run log retention**: Completed run logs pruned automatically

### Technical Debt / Concerns

1. **Timer persistence**: Timers are in-memory; system restart requires job state reload and timer recomputation
2. **Heartbeat coupling**: Job wake mode depends on session heartbeat system being active
3. **Delivery failure handling**: Best-effort delivery can silently fail; failureDestination provides alerting but adds complexity
4. **mcporter external dependency**: MCP integration requires separate mcporter binary installation

---

## Cross-Cutting Concerns

### Error Handling
- Failover errors with typed reasons, status codes, and error kinds
- Graceful degradation (e.g., sharp -> sips on macOS/Bun)
-ENOENT handling throughout (missing Docker, missing files)
- Abort signal propagation for cancellable operations

### Security Integration Points
1. External content wrapping for webhooks/email
2. Sandbox tool policy enforcement for cron job execution
3. Secret resolution for credential management
4. Gateway HTTP tool denylist for invoke endpoint

### Testing Patterns
- Colocated `*.test.ts` files next to source
- Large test files suggest complex behavior (audit.test.ts 125KB+)
- Test harnesses with dependency injection for mocking

---

## Dependencies Between Features

```
Security & Sandboxing
    |
    +-- Tool policy affects Cron job execution (sandbox tool allowlists)
    +-- Secrets required for webhook auth tokens
    +-- External content wrapping for hook payloads

Media Pipeline
    |
    +-- Media attached to cron job payloads
    +-- Image generation via agent tools
    +-- Media understanding for incoming attachments

Automation
    |
    +-- Cron jobs use sandbox execution
    +-- Webhooks trigger media fetch
    +-- MCP tools available via mcporter
```

---

## Files of Interest by Feature

### Security & Sandboxing
- `src/security/audit.ts` - Main audit engine
- `src/security/dm-policy-shared.ts` - Access control decisions
- `src/security/external-content.ts` - Prompt injection detection
- `src/security/dangerous-tools.ts` - Tool risk registry
- `src/security/fix.ts` - Auto-remediation
- `src/agents/sandbox/docker.ts` - Docker container execution
- `src/agents/sandbox/tool-policy.ts` - Per-agent tool permissions
- `src/secrets/configure.ts` - Secrets setup wizard
- `src/secrets/resolve.ts` - Secret ref resolution

### Media Pipeline
- `src/media/store.ts` - Media file lifecycle
- `src/media/image-ops.ts` - Image processing with sharp
- `src/media/web-media.ts` - Remote media fetching
- `src/media-understanding/apply.ts` - Media understanding pipeline
- `src/media-understanding/runner.ts` - Model orchestration
- `src/image-generation/runtime.ts` - Image generation with fallback

### Automation
- `src/cron/service.ts` - Cron service entry
- `src/cron/service/ops.ts` - CRUD operations
- `src/cron/service/timer.ts` - Timer management (largest file)
- `src/cron/service/jobs.ts` - Job state machine
- `src/cron/schedule.ts` - Cron parsing with croner
- `src/gateway/server-http.ts` - Webhook HTTP handlers
- `skills/mcporter/SKILL.md` - MCP CLI documentation
