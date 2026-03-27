# Features Batch 3: Lifecycle Hooks, Voice UI, Skills System

**Features:** 7, 8, 9
**Date:** 2026-03-26
**Repo:** /Users/sheldon/Documents/claw/reference/gitclaw

---

## Feature 7: Lifecycle Hooks System

### Overview

The Lifecycle Hooks System provides programmatic and shell-script-based hooks for gating, logging, and controlling agent behavior at key lifecycle points. Hooks can block tool execution, modify arguments, or run side effects at session start, tool use, response completion, and error conditions.

### Core Implementation Files

- **Primary:** `src/hooks.ts` (186 lines)
- **SDK variant:** `src/sdk-hooks.ts` (52 lines)
- **Integration:** `src/index.ts`, `src/sdk.ts`

### Architecture

#### Hook Types

Four hook points are defined in `HooksConfig`:

```
on_session_start  — fires once when agent session begins
pre_tool_use      — fires before each tool execution (can block/modify)
post_response     — fires after each agent response
on_error          — fires when an error occurs
```

#### HookDefinition Interface

```typescript
interface HookDefinition {
  script: string;           // Shell script path (relative to hooks/ dir)
  description?: string;     // Human-readable name
  baseDir?: string;         // Override base dir (for plugin hooks)
  _handler?: Function;      // Programmatic hook (in-process callback)
}
```

#### HookResult Interface

```typescript
interface HookResult {
  action: "allow" | "block" | "modify";
  reason?: string;          // Why blocked/modified
  args?: Record<string, any>; // Modified tool arguments (for modify action)
}
```

### Execution Model

**Two execution modes:**

1. **Programmatic hooks** (`_handler` field): Called directly in-process. Allows TypeScript callbacks without spawning processes.

2. **Shell hooks** (`script` field): Spawns a `sh` subprocess with the script path. Input is JSON-serialized to stdin, output is JSON from stdout.

**Path Traversal Protection:**
```typescript
const resolvedScript = resolve(scriptPath);
const allowedBase = resolve(baseDir);
if (!resolvedScript.startsWith(allowedBase + "/") && resolvedScript !== allowedBase) {
  reject(new Error(`Hook "${hook.script}" escapes its base directory`));
}
```

**Timeout:** 10 seconds hard limit, after which the process is killed with SIGTERM.

**JSON Contract:**
- Stdin: `JSON.stringify(input)` where input varies by hook type
- Stdout: JSON `HookResult` or non-JSON (treated as allow)

### Hook Input Structures

**pre_tool_use input:**
```json
{
  "event": "pre_tool_use",
  "session_id": "uuid",
  "tool": "tool_name",
  "args": { ... tool arguments ... }
}
```

**on_session_start input:**
```json
{ "event": "on_session_start", "session_id": "uuid" }
```

**post_response input:**
```json
{ "event": "post_response", "session_id": "uuid", "response": "..." }
```

**on_error input:**
```json
{ "event": "on_error", "session_id": "uuid", "error": "...", "tool": "tool_name" }
```

### Key Design Decisions

1. **Hook chaining with early termination**: `runHooks()` iterates through hook array, stopping on first `block` or `modify` result. `allow` continues to next hook.

2. **Hook errors do NOT block execution**: Errors are logged but `allow` is the default fallback. This prevents a buggy hook from halting the agent.

3. **Tool wrapping pattern**: `wrapToolWithHooks()` creates a proxy around `tool.execute()` that runs pre_tool_use hooks before delegating to the original implementation.

4. **Dual integration**: Hooks are integrated in both `src/index.ts` (CLI) and `src/sdk.ts` (SDK), ensuring consistent behavior across interfaces.

5. **Plugin hooks use baseDir override**: Plugins can specify their own `baseDir` to run hooks from within the plugin's directory structure rather than the agent's `hooks/` directory.

### Clever Solutions

1. **stdin/stdout JSON contract**: Shell hooks receive input via stdin and return JSON via stdout. This is clean but relies on the subprocess behaving correctly.

2. **Non-JSON fallback**: If stdout is not valid JSON, the hook is treated as `allow`. This makes simple echo hooks work without emitting JSON.

3. **Hook array order matters**: Since hooks chain and early-terminate on block/modify, the order in `hooks.yaml` is semantically significant.

### Technical Debt / Concerns

1. **10s timeout is hardcoded**: No configuration for long-running hooks. If legitimate operations need more time (e.g., network calls in hook scripts), they will be killed.

2. **No hook result caching**: Each tool execution triggers hook evaluation. For expensive hook scripts, there's no caching mechanism.

3. **Hooks.yaml discovery is optional**: `loadHooksConfig()` returns `null` if the file doesn't exist. This is silent - no warning that hooks directory is missing.

4. **No hook concurrency control**: If multiple tools fire simultaneously, their pre_tool_use hooks run in parallel (via Promise.all in the tool wrapper). This could cause race conditions if hooks modify shared state.

### Integration Points

In `src/index.ts`:
- Line 137: `post_response` hooks run after agent response
- Line 454: `on_session_start` hooks fire on session begin
- Line 521: Tools are wrapped with `wrapToolWithHooks()`
- Line 572, 722: `on_error` hooks fire on errors

In `src/sdk.ts`:
- Line 206-210: Tool wrapping with hooks
- Line 223: `on_session_start` hooks
- Line 367: `post_response` hooks

### Usage Pattern

Agent directory structure:
```
agent/
  hooks/
    hooks.yaml    # Hook definitions
    script1.sh    # Hook script
```

Example `hooks/hooks.yaml`:
```yaml
hooks:
  pre_tool_use:
    - script: verify.sh
      description: "Verify tool arguments"
  on_session_start:
    - script: welcome.sh
      description: "Session initialization"
```

---

## Feature 8: Voice UI

### Overview

Voice UI is a browser-based voice interface running at `localhost:3333`. It supports two backends: OpenAI Realtime API and Google Gemini Multimodal Live. The feature includes audio processing (24kHz/16kHz conversion), mood tracking, memory automation, session journaling, photo capture, Telegram integration, WhatsApp integration, and workflow execution.

### Core Implementation Files

| File | Purpose |
|------|---------|
| `src/voice/server.ts` | Main voice server (~1100+ lines) |
| `src/voice/adapter.ts` | Type definitions and `MultimodalAdapter` interface |
| `src/voice/openai-realtime.ts` | OpenAI Realtime API adapter |
| `src/voice/gemini-live.ts` | Google Gemini Live adapter |
| `src/voice/chat-history.ts` | Chat history persistence |
| `src/voice/ui.html` | Browser UI (115KB, large) |

### Architecture

#### Adapter Pattern

The `MultimodalAdapter` interface abstracts over OpenAI and Gemini:

```typescript
interface MultimodalAdapter {
  connect(opts: {
    toolHandler: (query: string) => Promise<string>;
    onMessage: (msg: ServerMessage) => void;
  }): Promise<void>;
  send(msg: ClientMessage): void;
  disconnect(): Promise<void>;
}
```

**ClientMessage types** (Browser → Server):
- `audio` — PCM audio data
- `video_frame` — Camera/screen frame
- `text` — Text input
- `file` — File attachment

**ServerMessage types** (Server → Browser):
- `audio_delta` — Audio response
- `transcript` — User/assistant transcript
- `agent_working` — Agent is processing
- `agent_done` — Agent finished
- `tool_call` / `tool_result` — Tool activity
- `error` — Error occurred
- `interrupt` — User interrupted

#### Audio Processing

**24kHz ↔ 16kHz conversion** is required because:
- Browser audio API: 24kHz
- Gemini: 16kHz
- OpenAI: handles internally

```typescript
// Gemini input: downsample 24k→16k
function downsample24kTo16k(base64_24k: string): string {
  const samples24 = new Int16Array(/* 24kHz data */);
  const outLength = Math.floor(samples24.length * 2 / 3);
  // Linear interpolation
  for (let i = 0; i < outLength; i++) {
    const srcIdx = i * 1.5;
    const lo = Math.floor(srcIdx);
    const frac = srcIdx - lo;
    const hi = Math.min(lo + 1, samples24.length - 1);
    samples16[i] = Math.round(samples24[lo] * (1 - frac) + samples24[hi] * frac);
  }
  return Buffer.from(samples16.buffer).toString("base64");
}
```

**Gemini output**: Upsample 16kHz → 24kHz for browser playback.

### OpenAI Realtime Adapter

Key behaviors:

1. **Dual auth strategy**: Tries direct WebSocket first. If auth fails, falls back to ephemeral token via REST API:
   ```typescript
   const sessionResp = await fetch("https://api.openai.com/v1/realtime/sessions", {
     method: "POST",
     headers: { "Authorization": `Bearer ${this.config.apiKey}` },
     body: JSON.stringify({ model }),
   });
   const ephemeralKey = session.client_secret.value;
   ```

2. **Server-side VAD**: Uses `server_vad` (Voice Activity Detection) with threshold 0.6, prefix padding 400ms, silence duration 800ms.

3. **Video frame handling**: OpenAI doesn't support continuous video. Latest frame is stored and injected as an image on the next user turn via `conversation.item.create`.

4. **Speech interruption**: On `speech_started` event:
   - Sets `interrupted = true`
   - Injects latest video frame
   - Sends `response.cancel`
   - Emits `interrupt` to UI

5. **Whisper transcription**: `input_audio_transcription: { model: "whisper-1" }` transcribes user speech server-side.

### Gemini Live Adapter

Key behaviors:

1. **Continuous video**: Gemini supports native continuous video streaming. Each frame is sent as a `mediaChunk`.

2. **Tool calling**: Single `run_agent` function that delegates to the gitclaw agent.

3. **Context window compression**: Configured for 25000 trigger tokens with 12500 token sliding window.

4. **Audio output**: 16kHz output upsampled to 24kHz for browser.

### Voice Server (Main Hub)

The `startVoiceServer()` function in `server.ts` is the central orchestration point:

**Key subsystems:**

1. **Memory automation** (`MEMORY_PATTERNS`): Server-side regex patterns detect when users share personal information. If matched, triggers background memory save without waiting for the LLM to decide.

2. **Moment detection** (`MOMENT_PATTERNS`): Patterns like "haha", "lol", "love it", "that's amazing" trigger photo capture.

3. **Mood tracking**: Tracks mood distribution across 5 categories (happy, frustrated, curious, excited, calm) using regex patterns. Writes mood log to `memory/mood.md` at session end.

4. **Session journaling**: Uses the agent itself to generate journal entries reflecting on the conversation. Stored in `memory/journal/{date}.md`.

5. **Vitals snapshot**: Centralized CPU/memory/heap/uptime tracking with 1-second cache. Delta-based CPU calculation.

6. **Approval gates**: For workflow execution, can request approval via Telegram or WhatsApp before proceeding. 5-minute timeout.

7. **File watching**: Snapshots file tree before/after agent runs, detects new/modified files via `diffSnapshots()`.

### Clever Solutions

1. **Background memory save**: Pattern matching happens server-side, not relying on LLM to remember to save. "Fire and forget" pattern with `query()` iterator drain.

2. **Video frame injection on speech start**: Rather than sending frames continuously (which OpenAI doesn't support), the frame is captured at speech start - capturing what the user was looking at when they started talking.

3. **Broadcast to all browser clients**: `broadcastToBrowsers()` sends to all connected WebSocket clients. Allows multiple browser tabs to see the same voice session.

4. **Telegram long polling with offset tracking**: Custom polling implementation (not using `node-telegram-bot-api`) with update offset tracking for deduplication.

5. **Multipart form file uploads**: Telegram file sending uses raw multipart form construction rather than a library.

6. **.env loading without overwrite**: `loadEnvFile()` only sets vars that aren't already in `process.env`.

7. **Vitals delta-based CPU calculation**:
   ```typescript
   const userDelta = currentCpu.user - _lastCpuUsage.user;
   const sysDelta = currentCpu.system - _lastCpuUsage.system;
   const wallDeltaUs = Number(currentTime - _lastCpuTime) / 1000;
   const cpuPercent = wallDeltaUs > 0 ? Math.min(100, Math.round((userDelta + sysDelta) / wallDeltaUs * 100)) : 0;
   ```

### Technical Debt / Concerns

1. **UI.html is 115KB**: Large embedded UI. If UI needs changes, this file is cumbersome to edit.

2. **Hardcoded port 3333**: No configuration option, though `VoiceServerOptions.port` suggests it should be overridable.

3. **Telegram polling implementation**: Custom long-polling rather than using a library. Needs proper reconnection/backoff logic.

4. **WhatsApp uses `any` type**: `whatsappSock: any` — no typed WhatsApp client.

5. **Approval gate timeout hardcoded at 5 minutes**: `5 * 60 * 1000` — no configuration.

6. **Memory pattern matching is regex-only**: No learning or adjustment based on false positives.

7. **Git commits for mood/journal can fail silently**: Errors are swallowed with `/* saved even if commit fails */` comments.

8. **No connection cleanup on shutdown**: `pendingShutdownWork` array exists but work is not awaited on process exit.

9. **SkillFlow execution**: `executeFlow()` calls `query()` directly without hook integration.

### WebSocket/HTTP Server Setup

The server creates:
- HTTP server on configured port (default 3333)
- WebSocket server (`wss`) for browser connections
- Serves `ui.html` at root (`/`)
- REST endpoints for Telegram webhook
- File browser API (`GET /files`, `POST /files`, etc.)

### Composio Integration

If `COMPOSIO_API_KEY` is set, `ComposioAdapter` is instantiated and used for semantic tool search. The voice server can query Composio for relevant tools based on the user's prompt.

---

## Feature 9: Skills System

### Overview

Skills are composable instruction modules for agents. Each skill is a directory containing a `SKILL.md` file with YAML frontmatter defining name/description, plus optional `scripts/` and `references/` subdirectories. Skills are discovered from the agent's `skills/` directory and can be invoked via `/skill:name` slash commands.

### Core Implementation Files

- **Primary:** `src/skills.ts` (194 lines)
- **Related:** `src/workflows.ts` (SkillFlow definitions)

### Skill Discovery

```typescript
interface SkillMetadata {
  name: string;
  description: string;
  directory: string;
  filePath: string;
  confidence?: number;
  usage_count?: number;
  success_count?: number;
  failure_count?: number;
}
```

Discovery process (`discoverSkills()`):
1. Scans `agentDir/skills/` for directories or directory symlinks
2. For each entry, reads `SKILL.md` from within
3. Parses YAML frontmatter for `name` and `description`
4. Validates: name must match directory name, must be kebab-case
5. Sorts alphabetically by name
6. Returns `SkillMetadata[]`

### Frontmatter Validation

The `SKILL.md` format:
```markdown
---
name: example-skill
description: Example skill demonstrating the gitclaw skills system.
confidence: 0.9
usage_count: 42
---

# Skill Content Below
```

**Validation rules:**
1. `name` and `description` are required in frontmatter
2. `name` must match the directory name exactly
3. `name` must match kebab-case regex: `^[a-z0-9]+(-[a-z0-9]+)*$`
4. Missing or invalid frontmatter → skill is skipped (with warning)
5. Directory name mismatch → skill is skipped

### Skill Loading

```typescript
interface ParsedSkill extends SkillMetadata {
  instructions: string;  // Body content (markdown below frontmatter)
  hasScripts: boolean;    // Whether scripts/ directory exists
  hasReferences: boolean; // Whether references/ directory exists
}

async function loadSkill(meta: SkillMetadata): Promise<ParsedSkill> {
  const content = await readFile(meta.filePath, "utf-8");
  const { body } = parseFrontmatter(content);
  return {
    ...meta,
    instructions: body.trim(),
    hasScripts: await dirExists(join(meta.directory, "scripts")),
    hasReferences: await dirExists(join(meta.directory, "references")),
  };
}
```

### Skill Invocation

**Slash command expansion** (`expandSkillCommand()`):
```typescript
// Input: "/skill:example-skill arg1 arg2"
// Output: {
//   expanded: "<skill name=\"example-skill\" baseDir=\"...\">\n...",
//   skillName: "example-skill"
// }
```

Pattern: `/skill:([a-z0-9-]+)\s*([\s\S]*)$`

The expanded skill is injected into the prompt as:
```xml
<skill name="example-skill" baseDir="/path/to/skills/example-skill">
References are relative to /path/to/skills/example-skill.

[SKill.md body content]
</skill>
You MUST follow the skill instructions above. Do NOT use general alternatives.
```

### Prompt Formatting

`formatSkillsForPrompt()` generates a mandatory-skills-first prompt section:

```
# Skills — FIRST PRIORITY (MANDATORY)

CRITICAL: You have installed skills that provide specialized capabilities.
Before attempting ANY task — simple or complex — you MUST check if an installed skill handles it.

## Rules (MUST follow in order)
1. ALWAYS scan the skill list below BEFORE taking ANY action on a user request
2. If a skill's description matches or partially matches the task, you MUST load its full
   instructions using the `read` tool: `skills/<name>/SKILL.md` — do this BEFORE anything else
3. Follow the loaded skill instructions EXACTLY — do NOT improvise or use alternative approaches
4. NEVER use general-purpose workarounds when a skill provides the right tool
5. If multiple skills could apply, load the most specific one first
6. Even for seemingly simple tasks, CHECK SKILLS FIRST

## Enforcement
- If you skip checking skills and use a raw approach for a task that a skill handles,
  this is considered a FAILURE.

<available_skills>
<skill>
<name>example-skill</name>
<description>Example skill demonstrating the gitclaw skills system.</description>
<location>skills/example-skill/SKILL.md</location>
</skill>
</available_skills>

To load a skill's full instructions: read `skills/<name>/SKILL.md`
Scripts within a skill are relative to the skill's directory: `skills/<name>/scripts/`
```

### Skill Structure

Example: `skills/example-skill/`:
```
example-skill/
  SKILL.md          # Required: name, description frontmatter + instructions
  scripts/          # Optional: executable scripts
    hello.sh
  references/       # Optional: reference documents
    README.md
```

### Learning System Integration

Skills support optional learning metadata:
- `confidence`: 0.0-1.0, indicates reliability
- `usage_count`: Number of times skill was used
- `success_count`: Successful executions
- `failure_count`: Failed executions

These are parsed from frontmatter and stored in `SkillMetadata`. The learning system (Feature 17) presumably updates these based on execution results.

### Symlink Support

Skills can be symlinks to directories (useful for referencing skills in parent agents or external sources):
```typescript
// For symlinks, verify the target is actually a directory
if (entry.isSymbolicLink() && !(await dirExists(skillDir))) continue;
```

### Key Design Decisions

1. **Kebab-case naming enforced**: Only lowercase letters and numbers, hyphens allowed. Prevents filesystem issues and ensures consistent naming.

2. **Name must match directory**: The skill name in frontmatter MUST match its directory name. This prevents confusion when referencing skills by path.

3. **Skills are opt-in per conversation**: The agent is reminded to check skills via the prompt section, but there's no forced enforcement mechanism at the framework level. The "MUST" is in the prompt, not in code.

4. **Skill body is markdown instructions**: The content below frontmatter is the actual skill instructions. Can include markdown formatting, code blocks, etc.

5. **Scripts are skill-internal**: Scripts referenced in skill instructions are relative to the skill's directory, not the agent's working directory.

### Example Skills

**`skills/example-skill/SKILL.md`**:
```markdown
---
name: example-skill
description: Example skill that demonstrates the gitclaw skills system.
---

# Example Skill

This is a demo skill showing how gitclaw skills work.

## Usage

Run the hello script:
```bash
bash scripts/hello.sh
```
```

**`skills/gmail-email/SKILL.md`**:
```markdown
---
name: gmail-email
description: Send emails via Gmail SMTP using App Password authentication.
---

# Gmail Email Skill

Send emails via Gmail SMTP.

## Setup
1. Enable 2FA on Gmail
2. Generate App Password
3. Configure GMAIL_USER and GMAIL_APP_PASSWORD env vars

## Usage
python3 scripts/send_email.py --to "..." --subject "..." --body "..."
```

### Workflow Integration (SkillFlows)

Skills are chained together in `SkillFlows` (defined in `workflows.ts`):

```typescript
interface SkillFlowStep {
  skill: string;      // Skill name to invoke
  prompt: string;     // User prompt (with {input} placeholder)
  channel?: string;   // Approval channel (telegram/whatsapp)
}
```

The voice server's `executeFlow()` runs steps sequentially, loading each skill via `/skill:{step.skill}` and piping context between steps.

### Clever Solutions

1. **Symlink support for skill sharing**: Skills can be symlinks, allowing a single skill definition to be shared across multiple agents or referenced from parent directories.

2. **Frontmatter parsing with yaml library**: Uses `js-yaml` for robust YAML parsing, with fallback to raw content if no frontmatter exists.

3. **Skill directory validation**: Checks `isDirectory()` and `isSymbolicLink()` separately, then verifies symlink targets are actually directories.

### Technical Debt / Concerns

1. **No skill versioning**: If a skill changes, running sessions don't see updates until restarted.

2. **No skill dependencies**: Skills can't declare dependencies on other skills. Composition is implicit via SkillFlows.

3. **Skill loading reads from disk every time**: No caching of loaded skills. Each invocation reads and parses the SKILL.md file.

4. **No built-in skill discovery from URLs**: Unlike agents (which can extend from git URLs), skills must be local directories.

5. **Confidence/usage stats not automatically updated**: The fields exist in the type but nothing in `skills.ts` actually updates them. Presumably handled by the learning system (Feature 17).

6. **Skill instructions are just concatenated into prompt**: No sandboxing or security boundary between skills. A malicious skill could inject into the prompt.

---

## Cross-Feature Observations

### Hooks + Skills Integration
- Skills can be invoked from hooks (shell hooks can call `/skill:...` expansions)
- But there's no direct hook integration with SkillFlow execution

### Voice + Skills Integration
- Voice server uses `expandSkillCommand()` to handle `/skill:` commands in voice input
- SkillFlows can be triggered via `@flow_name` syntax in voice chat
- Voice UI has "Skills" panel showing discovered skills

### Voice + Hooks Integration
- Voice server doesn't directly invoke hooks
- Hooks operate at the agent/tool level, not at the voice server level
- The `toolHandler` in voice server is the `query()` call, which goes through SDK → hooks

### Skills + Workflows
- `SkillFlowDefinition` chains multiple skills together
- Each step invokes a skill via `/skill:{name}` expansion
- Flow execution runs skills sequentially, passing context via `runningContext`

---

## Summary Table

| Aspect | Lifecycle Hooks | Voice UI | Skills System |
|--------|-----------------|----------|---------------|
| **Entry point** | `src/hooks.ts` | `src/voice/server.ts` | `src/skills.ts` |
| **Size** | 186 lines | 1100+ lines | 194 lines |
| **Key interface** | `HookDefinition`, `HookResult` | `MultimodalAdapter` | `SkillMetadata`, `ParsedSkill` |
| **Execution mode** | Shell scripts + in-process | WebSocket server | Prompt expansion |
| **Configuration** | `hooks/hooks.yaml` | `COMPOSIO_API_KEY`, `.env` | `skills/*/SKILL.md` |
| **Extensibility** | Plugin hooks via `baseDir` | Multiple adapter backends | SkillFlow chaining |
| **Security** | Path traversal guard, 10s timeout | Path traversal guard, allowed users | Skill instructions in prompt |
| **Notable patterns** | Early termination chain | Delta CPU, video injection | Kebab-case validation |
