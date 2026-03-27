# Security: GitClaw

## Secrets Management

### Environment Variables

API tokens and secrets should be provided via environment variables:

- `OPENAI_API_KEY` -- OpenAI access
- `GITHUB_TOKEN` / `GIT_TOKEN` -- GitHub PAT for repo mode
- `GITCLAW_*` -- Any custom environment variables

### Memory Exclusion Rule

From `RULES.md`:

> **No secrets in memory.** Never store API keys, passwords, tokens, or credentials in MEMORY.md.

### Token Resolution Order

In `sdk.ts` for repo mode:

```typescript
const token = options.repo.token
  || process.env.GITHUB_TOKEN
  || process.env.GIT_TOKEN;
```

### Config Environment Interpolation

Plugin configuration supports env var interpolation:

```yaml
plugins:
  my-plugin:
    config:
      api_key: "${MY_API_KEY}"  # Supports env interpolation
```

## Sandbox Isolation

### VM-Based Execution

Sandbox mode runs agents in isolated VM via e2b provider:

```typescript
export interface SandboxConfig {
  provider: "e2b";
  template?: string;
  timeout?: number;
  repository?: string;
  token?: string;
  session?: string;
  autoCommit?: boolean;
  envs?: Record<string, string>;
}
```

### GitMachine Integration

Sandbox uses `gitmachine` package for isolated git operations:

```typescript
const gitMachine = new gitMachine.GitMachine({
  provider: config.provider,
  template: config.template,
  timeout: config.timeout,
  repository,
  token,
  session: config.session,
  autoCommit: config.autoCommit ?? true,
  envs: config.envs,
});
```

### Sandbox vs Repo Mutually Exclusive

```typescript
if (options.repo && options.sandbox) {
  throw new Error("repo and sandbox options are mutually exclusive");
}
```

## Hook-Based Security Gates

### Pre-Tool-Use Hooks

Block or modify dangerous operations before execution:

```typescript
preToolUse: async (ctx) => {
  // Block destructive commands
  if (ctx.toolName === "cli" && ctx.args.command?.includes("rm -rf"))
    return { action: "block", reason: "Destructive command blocked" };

  // Modify arguments for safety
  if (ctx.toolName === "write" && !ctx.args.path.startsWith("/safe/"))
    return { action: "modify", args: { ...ctx.args, path: `/safe/${ctx.args.path}` } };

  return { action: "allow" };
}
```

### Hook Script Interface

Script-based hooks receive context as JSON on stdin and return:

```json
{ "action": "allow" }
{ "action": "block", "reason": "Not permitted" }
{ "action": "modify", "args": { "modified": "args" } }
```

### Hook Types

- `on_session_start` -- Validate environment before session
- `pre_tool_use` -- Gate/audit tool calls
- `post_response` -- Notify or log after responses
- `on_error` -- Handle errors

## Compliance Framework

### Risk Levels

Agents can specify risk levels:

```yaml
compliance:
  risk_level: high  # low | medium | high | critical
  human_in_the_loop: true
```

### Validation Rules

Compliance module enforces:

1. **High/Critical risk** requires `human_in_the_loop`
2. **Critical risk** must have `audit_logging` enabled
3. **Regulatory frameworks** require `recordkeeping` configuration
4. **Audit logging** requires `retention_days` policy

```typescript
export function validateCompliance(manifest: AgentManifest): ComplianceWarning[] {
  // Returns warnings/errors for rule violations
}
```

### Audit Logging

Full tool invocation traces written to `.gitagent/audit.jsonl`:

```typescript
// Audit log format per event:
// - tool_use events with args
// - tool_result events with output
// - session metadata
// - errors
```

## Tool Allowlists/Denylists

### Allowed Tools

```typescript
for await (const msg of query({
  prompt: "...",
  allowedTools: ["read", "cli"],  // Only these tools
})) {}
```

### Disallowed Tools

```typescript
for await (const msg of query({
  prompt: "...",
  disallowedTools: ["cli"],  // Block dangerous tools
})) {}
```

### Replace Builtin Tools

```typescript
for await (const msg of query({
  prompt: "...",
  replaceBuiltinTools: true,  // Skip cli/read/write/memory
  tools: [customTool],         // Only custom tools
})) {}
```

## Path Resolution Safety

Sandbox paths are validated and resolved:

```typescript
export function resolveSandboxPath(path: string, repoRoot: string): string {
  if (path.startsWith("~/") || path === "~") {
    path = homedir() + path.slice(1);
  }
  if (path.startsWith("/")) return path;
  return repoRoot.endsWith("/") ? repoRoot + path : repoRoot + "/" + path;
}
```

## Resource Limits

Built-in tools enforce limits to prevent resource exhaustion:

```typescript
export const MAX_OUTPUT = 100_000;  // ~100KB max output to LLM
export const MAX_LINES = 2000;      // Max lines per read
export const MAX_BYTES = 100_000;   // Max bytes per read
export const DEFAULT_TIMEOUT = 120; // CLI timeout in seconds
```

## CLI Safety

Destructive commands require explicit confirmation:

> **No destructive commands without confirmation.** Commands like `rm -rf`, `git reset --hard`, `git push --force`, or anything that deletes data require explicit user approval.

## Agent Directory Scoping

Agents should only operate within their directory:

> **Stay in scope.** Only operate within the current repository unless explicitly asked to go elsewhere.

## Dependency Considerations

### Peer Dependency Optionality

`gitmachine` is an optional peer dependency:

```json
"peerDependencies": {
  "gitmachine": ">=0.1.0"
},
"peerDependenciesMeta": {
  "gitmachine": {
    "optional": true
  }
}
```

### Dynamic Import for Optional Dependencies

Sandbox module uses dynamic import:

```typescript
try {
  gitmachine = await import("gitmachine");
} catch {
  throw new Error(
    "Sandbox mode requires the 'gitmachine' package.\n" +
    "Install it with: npm install gitmachine",
  );
}
```

## Error Reporting

Errors are surfaced honestly rather than silently retried:

> **Report errors honestly.** If a command fails or produces unexpected output, report it rather than silently retrying.
