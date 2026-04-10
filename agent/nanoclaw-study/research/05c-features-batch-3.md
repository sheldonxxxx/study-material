# NanoClaw Feature Batch 3: Features 7-9 Deep Dive

**Research Date:** 2026-03-26
**Repo:** `/Users/sheldon/Documents/claw/reference/nanoclaw`

---

## Feature 7: Credential Security (Agent Vault)

### Overview

NanoClaw implements a "never trust the container" security model where API keys and tokens never enter containers. Instead, credentials are managed by either:

1. **OneCLI Agent Vault** (default) - External gateway service that intercepts HTTPS traffic and injects credentials at request time
2. **Native Credential Proxy** (optional skill) - Built-in proxy reading from `.env`

### Core Implementation

#### 1. Environment Variable Isolation (`src/env.ts`)

The `readEnvFile()` function is the gateway for all credential access:

```typescript
export function readEnvFile(keys: string[]): Record<string, string> {
  const envFile = path.join(process.cwd(), '.env');
  // ...
  // Does NOT load into process.env — callers decide what to do
  // This keeps secrets out of the process environment so they don't leak to child processes
}
```

**Key insight:** Values are returned to callers, never put into `process.env`. This prevents child processes (containers) from inheriting secrets via environment.

#### 2. OneCLI Configuration (`src/config.ts`)

```typescript
const envConfig = readEnvFile(['ONECLI_URL']);
export const ONECLI_URL =
  process.env.ONECLI_URL || envConfig.ONECLI_URL || 'http://localhost:10254';
```

The OneCLI URL is the only credential-related value that CAN be in `process.env` since it's not a secret.

#### 3. Container Credential Injection (`src/container-runner.ts`)

The `buildContainerArgs()` function applies OneCLI gateway configuration to container startup:

```typescript
// OneCLI gateway handles credential injection — containers never see real secrets.
// The gateway intercepts HTTPS traffic and injects API keys or OAuth tokens.
const onecliApplied = await onecli.applyContainerConfig(args, {
  addHostMapping: false, // Nanoclaw already handles host gateway
  agent: agentIdentifier,
});
if (onecliApplied) {
  logger.info({ containerName }, 'OneCLI gateway config applied');
} else {
  logger.warn(
    { containerName },
    'OneCLI gateway not reachable — container will have no credentials',
  );
}
```

**How it works:** OneCLI's `applyContainerConfig()` modifies the Docker arguments to configure the gateway. The gateway intercepts outbound HTTPS requests and injects credentials based on host patterns.

#### 4. .env Shadowing for Main Group (`src/container-runner.ts`)

For the privileged main group, `.env` is shadowed with `/dev/null`:

```typescript
if (isMain) {
  // Shadow .env so the agent cannot read secrets from the mounted project root.
  // Credentials are injected by the OneCLI gateway, never exposed to containers.
  const envFile = path.join(projectRoot, '.env');
  if (fs.existsSync(envFile)) {
    mounts.push({
      hostPath: '/dev/null',
      containerPath: '/workspace/project/.env',
      readonly: true,
    });
  }
}
```

#### 5. Mount Security Blocks Sensitive Paths (`src/mount-security.ts`)

The mount allowlist explicitly blocks credential-adjacent paths:

```typescript
const DEFAULT_BLOCKED_PATTERNS = [
  '.ssh',
  '.gnupg',
  '.aws',
  '.azure',
  '.gcloud',
  '.kube',
  '.docker',
  'credentials',  // <-- blocks any dir named "credentials"
  '.env',        // <-- blocks .env files
  '.netrc',
  // ... more
];
```

### OneCLI Initialization Skill (`.claude/skills/init-onecli/`)

The `/init-onecli` skill handles installation and migration:

**Phase 1: Pre-flight**
- Checks if OneCLI is already working (`onecli version` + health check)
- Checks for native credential proxy and offers choice

**Phase 2: Install**
```bash
curl -fsSL onecli.sh/install | sh
curl -fsSL onecli.sh/cli/install | sh
onecli config set api-host http://127.0.0.1:10254
```

**Phase 3: Migrate credentials**
- Scans `.env` for `ANTHROPIC_API_KEY`, `CLAUDE_CODE_OAUTH_TOKEN`, `ANTHROPIC_AUTH_TOKEN`
- Migrates to OneCLI vault with host patterns:
  ```bash
  onecli secrets create --name Anthropic --type anthropic --value <key> --host-pattern api.anthropic.com
  ```
- Removes migrated lines from `.env`
- Offers to migrate other container-facing credentials (OpenAI, Parallel, etc.)

**Phase 4: Verify**
- Checks logs for `OneCLI gateway config applied` messages
- Tests agent responds to messages

### Alternative: Native Credential Proxy (`.claude/skills/use-native-credential-proxy/`)

For users who don't want to install OneCLI, this skill:
- Merges `src/credential-proxy.ts` (not present in current codebase - likely from skill branch)
- Replaces OneCLI with a built-in HTTP proxy on port 3001 (configurable via `CREDENTIAL_PROXY_PORT`)
- Reads credentials directly from `.env`

The skill notes:
> "Containers get credentials injected via a local HTTP proxy that reads from `.env` — no external services needed."

### Security Model Summary

| Layer | Protection |
|-------|-----------|
| `env.ts` | Secrets stay out of `process.env` |
| `.env` shadowing | Main group's container can't read `.env` |
| OneCLI gateway | Intercepts HTTPS, injects at request time |
| Mount blocklist | Prevents mounting credential-adjacent paths |
| Per-group agents | OneCLI supports per-agent policies and rate limits |

---

## Feature 8: Web Access (Browser Automation)

### Overview

Agents running in containers can browse the web via the `agent-browser` CLI tool, which provides Playwright-based browser automation.

### Implementation

#### Container Setup (`container/Dockerfile`)

The container image includes Chromium and the `agent-browser` tool:

```dockerfile
# Install system dependencies for Chromium
RUN apt-get update && apt-get install -y \
    chromium \
    fonts-liberation \
    fonts-noto-cjk \
    fonts-noto-color-emoji \
    libgbm1 \
    libnss3 \
    libatk-bridge2.0-0 \
    libgtk-3-0 \
    # ... more X11/graphics libs
    && rm -rf /var/lib/apt/lists/*

# Set Chromium path for agent-browser
ENV AGENT_BROWSER_EXECUTABLE_PATH=/usr/bin/chromium
ENV PLAYWRIGHT_CHROMIUM_EXECUTABLE_PATH=/usr/bin/chromium

# Install agent-browser and claude-code globally
RUN npm install -g agent-browser @anthropic-ai/claude-code
```

#### Container Skill (`container/skills/agent-browser/SKILL.md`)

The skill is defined entirely in SKILL.md with no additional code - it documents the `agent-browser` CLI interface that agents use directly:

**Core workflow:**
```bash
agent-browser open <url>        # Navigate to page
agent-browser snapshot -i       # Get interactive elements with refs
agent-browser click @e1         # Click element by ref
agent-browser fill @e2 "text"   # Fill input by ref
agent-browser close             # Close browser
```

**Commands supported:**
- Navigation: `open`, `back`, `forward`, `reload`, `close`
- Snapshot: `snapshot [-i]` for interactive elements, `-c` compact, `-d N` depth limit
- Interactions: `click`, `dblclick`, `fill`, `type`, `press`, `hover`, `check`, `uncheck`, `select`, `scroll`
- Information: `get text`, `get html`, `get value`, `get attr`, `get title`, `get url`, `get count`
- Screenshots/PDF: `screenshot [--full]`, `pdf`
- Wait: `wait @e1`, `wait 2000`, `wait --text "..."`, `wait --url "..."`, `wait --load networkidle`
- Semantic locators: `find role button click --name "Submit"`
- Auth state: `state save/load auth.json`
- Cookies/Storage: `cookies`, `storage local`
- JavaScript: `eval`

**Authentication pattern:**
```bash
agent-browser open https://app.example.com/login
agent-browser state save auth.json  # After logging in once
# Later: load saved state
agent-browser state load auth.json
agent-browser open https://app.example.com/dashboard
```

### Key Implementation Details

1. **Container skill sync:** Skills from `container/skills/` are copied into each group's `.claude/skills/` at runtime (`src/container-runner.ts` lines 151-161)

2. **Non-blocking I/O:** Browser operations are synchronous CLI commands executed by the agent inside the container

3. **State persistence:** Auth state can be saved to JSON and reloaded, enabling session persistence across container restarts

### Architecture

```
Container (agent-browser CLI)
    |
    +---> Playwright (headless Chromium)
              |
              +---> Websites
```

The agent invokes `agent-browser` commands via `Bash(agent-browser:*)` tool (as declared in the skill frontmatter).

---

## Feature 9: Trigger Word Processing

### Overview

Messages are filtered by a trigger word (default: `@Andy`) to determine when the assistant should respond. This prevents the assistant from responding to every message in a group chat.

### Implementation

#### 1. Trigger Pattern Configuration (`src/config.ts`)

```typescript
export function buildTriggerPattern(trigger: string): RegExp {
  return new RegExp(`^${escapeRegex(trigger.trim())}\\b`, 'i');
}

export const DEFAULT_TRIGGER = `@${ASSISTANT_NAME}`;

export function getTriggerPattern(trigger?: string): RegExp {
  const normalizedTrigger = trigger?.trim();
  return buildTriggerPattern(normalizedTrigger || DEFAULT_TRIGGER);
}

export const TRIGGER_PATTERN = buildTriggerPattern(DEFAULT_TRIGGER);
```

**Pattern behavior:**
- `^` - Must be at start of message
- `\b` - Word boundary (prevents `@AndyExtra` matching)
- `i` - Case insensitive

**Test cases from `src/formatting.test.ts`:**
```typescript
TRIGGER_PATTERN.test(`@Andy hello`)     // true - start match
TRIGGER_PATTERN.test(`hello @Andy`)      // false - not at start
TRIGGER_PATTERN.test(`@Andyextra hello`) // false - word boundary
TRIGGER_PATTERN.test(`@Andy's thing`)    // true - apostrophe is word boundary
TRIGGER_PATTERN.test(`@Andy`)            // true - end of string is boundary
```

#### 2. Per-Group Trigger Configuration (`src/types.ts`)

```typescript
interface RegisteredGroup {
  // ...
  trigger: string;
  // ...
  isMain?: boolean; // True for main control group (no trigger, elevated privileges)
}
```

Each group can have a custom trigger. Groups are stored in SQLite with their trigger patterns.

#### 3. Sender Allowlist (`src/sender-allowlist.ts`)

Complements trigger filtering with sender-based access control:

```typescript
export interface ChatAllowlistEntry {
  allow: '*' | string[];  // '*' = everyone, or specific senders
  mode: 'trigger' | 'drop';  // 'trigger' = require trigger, 'drop' = drop all
}

export interface SenderAllowlistConfig {
  default: ChatAllowlistEntry;
  chats: Record<string, ChatAllowlistEntry>;  // per-chat overrides
  logDenied: boolean;
}
```

**Modes:**
- `trigger` (default): Require trigger AND sender must be allowed
- `drop`: Silently drop all messages from non-allowed senders

**Allowlist loading:**
- Stored at `~/.config/nanoclaw/sender-allowlist.json`
- Falls back to permissive defaults if missing

#### 4. Trigger Checking in Message Loop (`src/index.ts`)

**Message polling loop (lines 444-457):**
```typescript
// For non-main groups, only act on trigger messages.
// Non-trigger messages accumulate in DB and get pulled as
// context when a trigger eventually arrives.
if (needsTrigger) {
  const triggerPattern = getTriggerPattern(group.trigger);
  const allowlistCfg = loadSenderAllowlist();
  const hasTrigger = groupMessages.some(
    (m) =>
      triggerPattern.test(m.content.trim()) &&
      (m.is_from_me ||
        isTriggerAllowed(chatJid, m.sender, allowlistCfg)),
  );
  if (!hasTrigger) continue;
}
```

**Key behavior:** Non-trigger messages are stored in the database but don't trigger agent processing. They accumulate and get included as context when a trigger message arrives.

**Processing loop (lines 217-227):**
```typescript
if (!isMainGroup && group.requiresTrigger !== false) {
  const triggerPattern = getTriggerPattern(group.trigger);
  const allowlistCfg = loadSenderAllowlist();
  const hasTrigger = missedMessages.some(
    (m) =>
      triggerPattern.test(m.content.trim()) &&
      (m.is_from_me || isTriggerAllowed(chatJid, m.sender, allowlistCfg)),
  );
  if (!hasTrigger) return true;
}
```

#### 5. Trigger Modes

| Mode | Behavior |
|------|----------|
| `requiresTrigger: undefined` | Default - requires trigger |
| `requiresTrigger: true` | Explicitly requires trigger |
| `requiresTrigger: false` | Processes all messages |
| `isMain: true` | Never requires trigger, has elevated privileges |

#### 6. `is_from_me` Bypass

Messages from the assistant itself (`is_from_me: true`) bypass the trigger check. This allows the assistant to process its own messages without requiring a trigger, which is important for:
- Follow-up questions
- Continuation after tool use
- Scheduled task results piped to the assistant

### Flow Diagram

```
Message arrives
       |
       v
Is main group? --> Yes --> Process immediately
       |
       No
       |
       v
requiresTrigger === false? --> Yes --> Process immediately
       |
       No
       |
       v
Has trigger word AND sender allowed?
       |
       +-- Yes --> Pull all messages since lastAgentTimestamp
       |         (including non-trigger context)
       |         --> Enqueue for agent processing
       |
       +-- No --> Skip (messages stay in DB for later)
```

### Edge Cases

1. **Custom triggers with regex chars:** `getTriggerPattern('@C.L.A.U.D.E')` escapes dots, so only exact match works
2. **Accumulated context:** Non-trigger messages between triggers are preserved and sent together
3. **Self-messages:** Always processed regardless of trigger (allows follow-ups)
4. **Multiple trigger messages:** All messages since `lastAgentTimestamp` are processed together
5. **Empty trigger:** Falls back to `DEFAULT_TRIGGER`

---

## Cross-Feature Interactions

### Trigger + Allowlist + Credential Security

When a trigger message is processed:
1. Sender allowlist is checked first
2. If allowed, message is formatted and sent to container
3. Container has no credentials - OneCLI gateway injects them at HTTP time
4. Response is routed back through `formatOutbound()` which strips `<internal>` tags

### Browser + Trigger

When an agent uses browser automation:
1. Agent runs in container with `agent-browser` CLI
2. Browser traffic goes through OneCLI gateway (if configured) for credential injection
3. If using native proxy, proxy handles injection
4. Trigger filtering applies to the original message, not browser interactions

---

## Technical Debt and Observations

1. **OneCLI SDK coupling:** The `@onecli-sh/sdk` is imported directly. If OneCLI changes its API, this could break. The SDK handles `applyContainerConfig()` magic.

2. **Native credential proxy not in main branch:** The `src/credential-proxy.ts` mentioned in the skill doesn't exist in the current codebase - it's merged from a skill branch.

3. **Mount security allowlist location:** Stored at `~/.config/nanoclaw/mount-allowlist.json` - outside the project dir to prevent container agents from modifying it. But if the file doesn't exist, additional mounts are blocked (safe default).

4. **Browser skill is documentation-only:** The `container/skills/agent-browser/SKILL.md` is purely a usage guide. The actual `agent-browser` CLI is a separate npm package installed globally in the container.

5. **Trigger pattern word boundary:** The `\b` word boundary with apostrophe (`@Andy's`) is a clever edge case handling - JavaScript's `\b` treats apostrophe as a word character, so `@Andy` matches before `'s`.

6. **SQLite stores trigger patterns:** Trigger configuration persists in the `registered_groups` table, allowing per-group customization across restarts.

---

## File Reference Summary

| Feature | Key Files |
|---------|-----------|
| Credential Security | `src/env.ts`, `src/config.ts`, `src/container-runner.ts`, `src/mount-security.ts`, `.claude/skills/init-onecli/`, `.claude/skills/use-native-credential-proxy/` |
| Web Access | `container/Dockerfile`, `container/skills/agent-browser/SKILL.md` |
| Trigger Word | `src/config.ts`, `src/sender-allowlist.ts`, `src/index.ts`, `src/types.ts`, `src/formatting.test.ts` |
