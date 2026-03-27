# Feature 16: Comment Checker & Quality Hooks

## Feature Overview

**Feature Index Entry:** Comment Checker & Code Quality Hooks
**Priority:** SECONDARY
**Implementation Location:** `src/hooks/comment-checker/`, `src/hooks/thinking-block-validator/`, `src/hooks/edit-error-recovery/`, `src/hooks/write-existing-file-guard/`

This feature comprises four distinct hooks that work together to maintain code quality and prevent common AI agent mistakes:

1. **Comment Checker** - Reminds agents to reduce excessive comments after code writes
2. **Thinking Block Validator** - Validates thinking blocks to prevent API errors
3. **Edit Error Recovery** - Recovers from edit tool failures
4. **Write Existing File Guard** - Prevents overwrites without prior reads

---

## 1. Comment Checker (`src/hooks/comment-checker/`)

### Purpose
The `comment-checker` hook analyzes code after `write`, `edit`, `multiedit`, or `apply_patch` tool operations to detect excessive comments. It reminds agents to reduce verbosity while intelligently ignoring BDD tests, directives, and docstrings.

### Architecture

**Files (12 total):**
```
comment-checker/
├── index.ts           # Re-exports
├── hook.ts            # Main hook factory (createCommentCheckerHooks)
├── hook.apply-patch.test.ts
├── cli.ts             # CLI binary management and execution
├── cli.test.ts        # CLI tests
├── cli-runner.ts      # CLI invocation wrapper with concurrency control
├── downloader.ts      # Lazy binary download from GitHub Releases
├── pending-calls.ts    # TTL-based call tracking
├── types.ts           # TypeScript interfaces
└── [additional files]
```

### Core Implementation

**Binary Dependency:** Uses `@code-yeongyu/go-claude-code-comment-checker` (v0.4.1 fallback) - a Go-based CLI that analyzes code for comment patterns. Binary is lazy-downloaded from GitHub Releases on first use.

**Hook Points Used:**
- `tool.execute.before` - Registers pending calls with file paths and content
- `tool.execute.after` - Runs comment checker CLI on successful tool execution

**Concurrency Control:** The `cli-runner.ts` implements a semaphore pattern via `withCommentCheckerLock()` to prevent concurrent CLI invocations. Only one comment check runs at a time; concurrent calls are skipped.

**Pending Calls TTL:** Calls older than 60 seconds are cleaned up automatically via `startPendingCallCleanup()` running every 10 seconds.

### Key Flow

```
tool.execute.before:
  1. Check if tool is write/edit/multiedit/apply_patch
  2. Extract filePath from args (handles filePath, file_path, path variants)
  3. Register pending call with sessionID, timestamp, content

tool.execute.after:
  1. Skip if output indicates tool failure (error:, failed to, could not)
  2. Take pending call for this callID
  3. Run comment-checker CLI with JSON input (session_id, tool_name, tool_input)
  4. If CLI returns exit code 2, append warning message to output
```

### Configuration

```typescript
// src/config/schema/comment-checker.ts
export const CommentCheckerConfigSchema = z.object({
  /** Custom prompt to replace the default warning message */
  custom_prompt: z.string().optional(),
})
```

The `custom_prompt` allows users to override the default comment warning message, with `{{comments}}` placeholder for detected comments XML.

### Clever Solutions

1. **Lazy Binary Download:** Binary is only downloaded when first needed, not during plugin initialization
2. **Background Initialization:** `startBackgroundInit()` triggers download early while other init happens
3. **Multi-path Resolution:** Checks multiple locations: cached binary first, then npm package, then triggers download
4. **Timeout Protection:** 30-second timeout with SIGTERM/SIGKILL escalation prevents hung CLI processes
5. **Semaphore Pattern:** Prevents CLI concurrency issues while allowing sequential calls
6. **Apply Patch Support:** Special handling for `apply_patch` tool via metadata parsing

### Technical Debt / Shortcuts

1. **Exit Code Conventions:** Relies on CLI exit codes: 0=no comments, 2=comments found, other=error. No explicit error messages.
2. **JSON via Stdin:** CLI receives JSON input via stdin rather than CLI arguments, requiring spawn with `stdin: "pipe"`
3. **Comment Type Filtering:** Done by external CLI (`go-claude-code-comment-checker`), not by this hook

---

## 2. Thinking Block Validator (`src/hooks/thinking-block-validator/`)

### Purpose
Prevents "Expected thinking/redacted_thinking but found tool_use" API errors by validating and fixing message structure **before** sending to Anthropic API. This is a **proactive** solution versus reactive session recovery.

### Architecture

**Files (3 total):**
```
thinking-block-validator/
├── index.ts      # Re-exports
├── hook.ts       # Main hook factory (createThinkingBlockValidatorHook)
└── hook.test.ts  # Comprehensive test suite
```

### Core Implementation

**Hook Point:** `experimental.chat.messages.transform` - Called before messages are converted to ModelMessage format and sent to API.

**Key Logic:**

1. **Detect Anthropic-signed thinking blocks:** Only real `type: "thinking"` blocks with valid `signature` field trigger the hook. GPT `type: "reasoning"` blocks are excluded (no Anthropic signature).

2. **Signature Validation:**
```typescript
function isSignedThinkingPart(part: Part): part is SignedThinkingPart {
  const signature = (part as { signature?: unknown }).signature
  const synthetic = (part as { synthetic?: unknown }).synthetic
  return typeof signature === "string" && signature.length > 0 && synthetic !== true
}
```

3. **Fix Strategy:** If an assistant message has `tool_use` but doesn't start with a thinking block, prepend the most recent valid thinking block from message history.

4. **Critical Constraint:** Cannot inject synthetic thinking blocks - they would fail API signature validation. Only reuse existing signed blocks verbatim.

### Key Flow

```
experimental.chat.messages.transform:
  1. Skip if no Anthropic-signed thinking blocks in history
  2. For each assistant message:
     a. If has content parts but doesn't start with thinking:
        i. Find most recent signed thinking part from earlier messages
        ii. Prepend it to message.parts
     b. If no real thinking part available, skip injection entirely
```

### Edge Cases Handled

1. **GPT reasoning blocks (`type: "reasoning"`)** - Intentionally excluded, no Anthropic signature
2. **Synthetic blocks** - Blocks with `synthetic: true` flag are skipped
3. **Missing signature** - Blocks without valid signature are skipped
4. **No prior thinking** - If no signed thinking block exists, skips injection (downstream error preferable to guaranteed API rejection)
5. **Already has thinking** - Doesn't re-inject if message already starts with thinking

### Clever Solutions

1. **Signature-based Detection:** More reliable than model name checks - works for Claude, GPT with thinking variants, and future models
2. **Verbatim Reuse:** Reuses original Part object including signature, avoiding signature validation failures
3. **Proactive vs Reactive:** Runs BEFORE API call, preventing user-visible errors

### Technical Debt / Shortcuts

1. **No Block Creation:** Cannot create synthetic thinking blocks when none exist - would cause API rejection
2. **Backward Search:** Searches message history backwards for most recent thinking block (O(n) scan)

---

## 3. Edit Error Recovery (`src/hooks/edit-error-recovery/`)

### Purpose
Detects common Edit tool errors caused by AI mistakes and injects a recovery reminder to guide the agent toward corrective action.

### Architecture

**Files (3 total):**
```
edit-error-recovery/
├── index.ts      # Re-exports (also re-exports EDIT_ERROR_PATTERNS, EDIT_ERROR_REMINDER)
├── hook.ts       # Main hook factory (createEditErrorRecoveryHook)
└── index.test.ts # Comprehensive test suite
```

### Error Patterns Detected

```typescript
export const EDIT_ERROR_PATTERNS = [
  "oldString and newString must be different",  // Trying to "edit" to same content
  "oldString not found",                        // Wrong assumption about file content
  "oldString found multiple times",             // Ambiguous match, need more context
] as const
```

### Recovery Reminder

```typescript
export const EDIT_ERROR_REMINDER = `
[EDIT ERROR - IMMEDIATE ACTION REQUIRED]

You made an Edit mistake. STOP and do this NOW:

1. READ the file immediately to see its ACTUAL current state
2. VERIFY what the content really looks like (your assumption was wrong)
3. APOLOGIZE briefly to the user for the error
4. CONTINUE with corrected action based on the real file content

DO NOT attempt another edit until you've read and verified the file state.
`
```

### Key Flow

```
tool.execute.after:
  1. Check if tool is "edit" (case-insensitive)
  2. Check if output.output is a string
  3. Search for EDIT_ERROR_PATTERNS in output (lowercased)
  4. If found, append EDIT_ERROR_REMINDER to output
```

### Test Coverage

- Error patterns with/without "Error:" prefix
- Case-insensitive tool name detection
- Non-Edit tools ignored
- Successful outputs not modified
- Undefined output handling (MCP tools)

### Clever Solutions

1. **Lowercase Matching:** Uses `outputLower.includes(pattern.toLowerCase())` for case-insensitive detection
2. **Minimal Mutation:** Only appends reminder, doesn't modify error message itself
3. **Reference to Issue:** Comments reference `https://github.com/sst/opencode/issues/4718` for context

---

## 4. Write Existing File Guard (`src/hooks/write-existing-file-guard/`)

### Purpose
Prevents agents from accidentally overwriting existing files without first reading them. Forces proper workflow: read before write on existing files.

### Architecture

**Files (3 total):**
```
write-existing-file-guard/
├── index.ts      # Re-exports
├── hook.ts       # Main hook factory (createWriteExistingFileGuardHook)
└── index.test.ts # Comprehensive test suite (550 lines)
```

### Core Concepts

**Permission Model:**
- `tool.execute.before` on `read` grants write permission for that file to the session
- `tool.execute.before` on `write` consumes write permission for that file
- Permission is session-scoped and path-canonicalized

**Blocked Operations:**
- Write to existing file without `overwrite: true` flag
- Without prior `read` in same session
- Across session boundaries (other session's read doesn't grant permission)

**Allowed Operations:**
- Write to non-existing file
- Write with `overwrite: true` flag
- Write to `.sisyphus/**` files (internal task system)
- Write to files outside session directory
- Overwrite after same-session read (consumes permission)

### LRU Session Eviction

When sessions exceed `MAX_TRACKED_SESSIONS` (256), least recently used session is evicted to prevent memory growth.

```typescript
const evictLeastRecentlyUsedSession = (): void => {
  let oldestSessionID: string | undefined
  let oldestSeen = Number.POSITIVE_INFINITY
  for (const [sessionID, lastSeen] of sessionLastAccess.entries()) {
    if (lastSeen < oldestSeen) {
      oldestSeen = lastSeen
      oldestSessionID = sessionID
    }
  }
  // ... delete oldest session's permissions
}
```

### Path Normalization

```typescript
function toCanonicalPath(absolutePath: string): string {
  let canonicalPath = absolutePath
  if (existsSync(absolutePath)) {
    try {
      canonicalPath = realpathSync.native(absolutePath)  // Resolves symlinks
    } catch {
      canonicalPath = absolutePath
    }
  } else {
    // For non-existent paths, resolve directory first
    const absoluteDir = dirname(absolutePath)
    const resolvedDir = existsSync(absoluteDir) ? realpathSync.native(absoluteDir) : absoluteDir
    canonicalPath = join(resolvedDir, basename(absolutePath))
  }
  return normalize(canonicalPath)
}
```

### Session Cleanup via Event

```typescript
event: async ({ event }: { event: { type: string; properties?: unknown } }) => {
  if (event.type !== "session.deleted") return
  const sessionID = props?.info?.id
  if (!sessionID) return
  readPermissionsBySession.delete(sessionID)
  sessionLastAccess.delete(sessionID)
}
```

### Test Scenarios Covered (18 test cases)

1. Non-existing file write - allowed
2. Existing file without read - blocked
3. Same-session read then write - allowed (consumes permission)
4. Concurrent writes with one read permission - only first succeeds
5. Cross-session read permission - blocked
6. `overwrite: true` boolean/string - bypasses guard, strips arg
7. `overwrite: false/falsy` - blocked
8. Two sessions read, one writes - other session invalidated
9. `.sisyphus/**` files - always allowed
10. Path arg variants (filePath, path, file_path) - all work
11. Missing path arg - ignored safely
12. Non-read-write tools - don't grant permission
13. Relative read / absolute write - works
14. Outside session directory - allowed
15. Session deletion cleanup - permissions removed
16. Case-different paths - follows platform behavior
17. Symlink read / real path write - allowed
18. Path cap (1024) with LRU eviction

### Clever Solutions

1. **Path Canonicalization:** Uses `realpathSync.native` to resolve symlinks and normalize casing
2. **Permission Consumption:** Read permission is consumed on write, preventing repeated overwrites without re-read
3. **Cross-Session Invalidation:** When one session writes, other sessions' read permissions for that path are invalidated
4. **Argument Mutation:** Overwrites the `overwrite` key in output args to keep it hook-only (not passed to actual tool)
5. **Unref'd Interval:** Cleanup interval uses `.unref()` to not prevent process exit

### Technical Debt / Shortcuts

1. **MAX_TRACKED_PATHS_PER_SESSION (1024):** Trims oldest paths when exceeded, but this may cause unexpected permission loss
2. **Case Sensitivity Assumption:** Comment notes case-insensitive volumes handled, but logic differs per platform

---

## Integration Points

### Hook Registration

**Tool Guard Hooks** (`src/plugin/hooks/create-tool-guard-hooks.ts`):
```typescript
commentChecker: isHookEnabled("comment-checker")
  ? safeHook("comment-checker", () => createCommentCheckerHooks(pluginConfig.comment_checker))
  : null

writeExistingFileGuard: isHookEnabled("write-existing-file-guard")
  ? safeHook("write-existing-file-guard", () => createWriteExistingFileGuardHook(ctx))
  : null
```

**Transform Hooks** (`src/plugin/hooks/create-transform-hooks.ts`):
```typescript
thinkingBlockValidator: isHookEnabled("thinking-block-validator")
  ? safeCreateHook("thinking-block-validator", () => createThinkingBlockValidatorHook(), ...)
  : null
```

### Hook Points Summary

| Hook | Hook Point | Timing |
|------|-----------|--------|
| comment-checker | `tool.execute.before`, `tool.execute.after` | Before/after tool |
| thinking-block-validator | `experimental.chat.messages.transform` | Before API send |
| edit-error-recovery | `tool.execute.after` | After tool |
| write-existing-file-guard | `tool.execute.before`, `event` | Before tool, session events |

---

## Observations

### Architectural Patterns

1. **Safe Hook Factory:** All hooks use `safeCreateHook()` wrapper for error isolation
2. **Configuration-driven:** Hooks can be enabled/disabled via `isHookEnabled()` check
3. **Session-scoped State:** Some hooks maintain session state (write-existing-file-guard)
4. **TTL-based Cleanup:** Pending calls and other temporary state use time-based cleanup

### Quality vs. Convenience Tradeoffs

1. **Comment Checker is External:** Relies on Go binary for actual comment analysis - not TypeScript-native
2. **Thinking Block Validator is Conservative:** Won't inject synthetic blocks, accepts potential downstream error
3. **Write Guard is Strict:** Could be seen as limiting; `overwrite` flag provides escape hatch

### Potential Issues

1. **comment-checker:** No explicit error handling if CLI binary fails to download
2. **thinking-block-validator:** Only handles `thinking`/`redacted_thinking` types, not future variants
3. **edit-error-recovery:** Pattern matching is simple string inclusion, could have false positives
4. **write-existing-file-guard:** Path cap of 1024 paths per session could cause permission loss under heavy multi-file editing

---

## Related Features

- **Session Recovery** (`src/hooks/session-recovery/`) - Reactive version of thinking block fixes
- **Preemptive Compaction** (`src/hooks/preemptive-compaction.ts`) - Context management
- **Hashline Edit** (`src/tools/hashline-edit/`) - Edit tool with hash validation
