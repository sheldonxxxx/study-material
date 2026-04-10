# OpenCode Security

## Overview

OpenCode is an AI-powered coding assistant that runs locally on the user's machine. The project takes a pragmatic approach to security, explicitly acknowledging that the permission system is a **UX feature rather than a security boundary**.

**Key Philosophy:** No sandbox is implemented. Users who need isolation should use Docker or VMs.

---

## Secrets Management

### Environment Variables

All secrets are managed through environment variables parsed via the `Flag` namespace in `packages/opencode/src/flag/flag.ts`:

```typescript
export const OPENCODE_SERVER_PASSWORD = process.env["OPENCODE_SERVER_PASSWORD"]
export const OPENCODE_SERVER_USERNAME = process.env["OPENCODE_SERVER_USERNAME"]
export const OPENCODE_DB = process.env["OPENCODE_DB"]
```

### File Permissions

Sensitive files are written with restricted permissions (0o600):

```typescript
// packages/opencode/src/auth/index.ts:78
yield* Effect.tryPromise({
  try: () => Filesystem.writeJson(file, { ...data, [norm]: info }, 0o600),
})

// packages/opencode/src/mcp/auth.ts:82
yield* fs.writeJson(filepath, { ...data, [mcpName]: entry }, 0o600)
```

### Auth Storage

Provider credentials are stored in JSON files:

| File | Contents | Location |
|------|----------|----------|
| `auth.json` | Provider API keys, OAuth tokens | `Global.Path.data` |
| `mcp-auth.json` | MCP server authentication | `Global.Path.data` |

---

## Authentication

### Server Mode Basic Auth

HTTP Basic Auth is available for server mode when `OPENCODE_SERVER_PASSWORD` is set:

```typescript
// packages/opencode/src/server/server.ts:89-93
const password = Flag.OPENCODE_SERVER_PASSWORD
if (!password) return next()
const username = Flag.OPENCODE_SERVER_USERNAME ?? "opencode"
return basicAuth({ username, password })(c, next)
```

### OAuth + API Key Support

The `Auth` namespace supports multiple authentication types:

```typescript
// packages/opencode/src/auth/index.ts
export class Oauth extends Schema.Class<Oauth>("OAuth")({
  type: Schema.Literal("oauth"),
  refresh: Schema.String,
  access: Schema.String,
  expires: Schema.Number,
})

export class Api extends Schema.Class<Api>("ApiAuth")({
  type: Schema.Literal("api"),
  key: Schema.String,
})
```

### MCP OAuth Authentication

MCP servers support OAuth 2.0 with PKCE via `McpAuth` namespace (`packages/opencode/src/mcp/auth.ts`):

- Token storage with expiry tracking
- Code verifier for PKCE
- OAuth state for CSRF protection

### Provider Auth Hooks

The `ProviderAuth` system allows plugins to define custom auth methods:

- OAuth flows with prompts
- API key validation

---

## Permission System

### Permission Model

The permission system is explicitly a **UX feature, not a security boundary**:

```typescript
// packages/opencode/src/permission/index.ts
export const Action = z.enum(["allow", "deny", "ask"])
export const Rule = z.object({
  permission: z.string(),
  pattern: z.string(),
  action: Action,
})
```

### Permission Evaluation

Rules are evaluated against patterns with wildcard matching:

```typescript
// packages/opencode/src/permission/evaluate.ts
export function evaluate(permission: string, pattern: string, ...rulesets: Ruleset[]): Rule
```

### File Path Boundaries

The `assertExternalDirectory` function checks if file operations stay within project boundaries:

```typescript
// packages/opencode/src/tool/external-directory.ts
export async function assertExternalDirectory(ctx: Tool.Context, target?: string, options?: Options) {
  if (Instance.containsPath(target)) return
  // Prompts for external directory access
  await ctx.ask({
    permission: "external_directory",
    patterns: [glob],
  })
}
```

---

## Input Validation

### Zod Schema Validation

All tool parameters use Zod for validation:

```typescript
// packages/opencode/src/tool/write.ts
parameters: z.object({
  content: z.string().describe("The content to write to the file"),
  filePath: z.string().describe("The absolute path to the file"),
})
```

### Tool Execution Validation

Tools are validated before execution:

```typescript
// packages/opencode/src/tool/tool.ts
toolInfo.execute = async (args, ctx) => {
  try {
    toolInfo.parameters.parse(args)
  } catch (error) {
    if (error instanceof z.ZodError && toolInfo.formatValidationError) {
      throw new Error(toolInfo.formatValidationError(error), { cause: error })
    }
  }
}
```

### Batch Tool Validation

```typescript
// packages/opencode/src/tool/batch.ts:62
const validatedParams = tool.parameters.parse(call.parameters)
```

---

## Content Security Policy

CSP headers are dynamically generated for the proxy endpoint:

```typescript
// packages/opencode/src/server/server.ts:53-54
const csp = (hash = "") =>
  `default-src 'self'; script-src 'self' 'wasm-unsafe-eval'${hash ? ` 'sha256-${hash}'` : ""}; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; font-src 'self' data:; media-src 'self' data:; connect-src 'self' data:`
```

---

## CORS Configuration

CORS is restrictive, allowing only specific origins:

```typescript
// packages/opencode/src/server/server.ts:113-134
origin(input) {
  if (input.startsWith("http://localhost:")) return input
  if (input.startsWith("http://127.0.0.1:")) return input
  if (input === "tauri://localhost" || input === "http://tauri.localhost" || input === "https://tauri.localhost")
    return input
  if (/^https:\/\/([a-z0-9-]+\.)*opencode\.ai$/.test(input)) return input
  if (opts?.cors?.includes(input)) return input
  return
}
```

**Allowed Origins:**
- `localhost:*` (HTTP)
- `127.0.0.1:*` (HTTP)
- `tauri://localhost` (Desktop)
- `*.opencode.ai` (Production domains)
- Custom origins via `opts?.cors`

---

## Session Sharing

Share URLs use a secret token:

```typescript
// packages/opencode/src/share/share.sql.ts
export const SessionShareTable = sqliteTable("session_share", {
  session_id: text().primaryKey().references(() => SessionTable.id, { onDelete: "cascade" }),
  id: text().notNull(),
  secret: text().notNull(),  // Secret for share URL
  url: text().notNull(),
})
```

---

## Threat Model

### Out of Scope (Explicit)

- **Server access** when server mode is enabled
- **Sandbox escapes** (no sandbox implemented)
- **LLM provider data handling**
- **MCP server behavior**
- **Malicious config files** (user-controlled)

### Security Controls

| Control | Implementation |
|---------|---------------|
| File permissions | 0o600 for secrets |
| CORS restrictions | localhost + opencode.ai domains only |
| CSP headers | Dynamic with hash |
| Basic Auth | Opt-in for server mode |
| Path boundaries | External directory prompts |
| Input validation | Zod schemas on all tools |
| Secrets storage | Local filesystem only |

### Data Storage

**No encryption at rest** for local data. The project trusts the local filesystem permissions as the security boundary.

---

## Key Security Files

| File | Purpose |
|------|---------|
| `SECURITY.md` | Threat model and vulnerability reporting |
| `packages/opencode/src/auth/index.ts` | Provider auth (OAuth, API keys) |
| `packages/opencode/src/mcp/auth.ts` | MCP OAuth tokens |
| `packages/opencode/src/permission/index.ts` | Permission evaluation |
| `packages/opencode/src/tool/external-directory.ts` | Path boundary checks |
| `packages/opencode/src/server/server.ts` | CSP, CORS, basic auth |

---

## Summary

OpenCode implements security through layered controls:

1. **File-level:** Restricted permissions (0o600) for credential storage
2. **Transport-level:** CORS restrictions and CSP headers
3. **Access-level:** Opt-in Basic Auth for server mode
4. **Input-level:** Zod validation on all tool parameters
5. **UX-level:** Permission prompts for external directory access

The project is explicit that it does **not** provide sandboxing or security boundaries. Users requiring isolation must use external mechanisms (Docker, VMs, etc.).
