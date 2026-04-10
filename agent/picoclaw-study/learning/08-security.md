# PicoClaw Security

## Security Model Overview

PicoClaw is in **pre-v1.0 development** with active security hardening. The README explicitly notes:
> PicoClaw is in early rapid development. There may be unresolved security issues. Do not deploy to production before v1.0.

## Secrets Management

### Two-File Configuration

PicoClaw separates sensitive from non-sensitive configuration:

**`config.json`** - Non-sensitive settings:
- Model definitions (name, provider, endpoint)
- Channel configurations (webhook URLs, public settings)
- Agent defaults and behavior settings
- Memory and tool configurations

**`.security.yml`** - Sensitive data:
- API keys (model providers, web search)
- Channel tokens and secrets
- OAuth credentials and refresh tokens
- Database passwords

### Security Config Structure

```go
// pkg/config/security.go
type SecurityConfig struct {
    ModelList map[string]ModelSecurityEntry  // API keys per model
    Channels  *ChannelsSecurity               // Channel tokens
    Web       *WebToolsSecurity               // Search API keys
    Skills    *SkillsSecurity                // GitHub, ClawHub tokens
}
```

### Environment Variable Override

All security fields support environment variable overrides:

```go
type TelegramSecurity struct {
    Token string `yaml:"token,omitempty" env:"PICOCLAW_CHANNELS_TELEGRAM_TOKEN"`
}
```

Example `.env`:
```bash
# LLM Providers
OPENAI_API_KEY=sk-xxx
ANTHROPIC_API_KEY=sk-ant-xxx
GEMINI_API_KEY=xxx

# Channels
PICOCLAW_CHANNELS_TELEGRAM_TOKEN=123456:ABC...
PICOCLAW_CHANNELS_FEISHU_APP_SECRET=xxx
```

### Sensitive Data Filtering

The security config implements automatic redaction for logs:

```go
// SecurityConfig.SensitiveDataReplacer()
// Returns a strings.Replacer that replaces all sensitive values with [FILTERED]
// Used in logging to prevent accidental secret exposure
```

## Authentication Implementation

### OAuth 2.0 Flows

PicoClaw supports multiple OAuth flows:

**1. Browser-based OAuth** (`LoginBrowser`):
```go
// Opens system browser for authentication
// Polls for token exchange
cred, err := auth.LoginBrowser(cfg)
```

**2. Device Code Flow** (`LoginDeviceCode`) - for headless environments:
```go
// Displays code for user to enter at provider's device flow URL
// Polls until user completes authentication
cred, err := auth.LoginDeviceCode(cfg)
```

**3. Setup Token** - Anthropic-specific:
```go
// Accepts pre-generated setup token from `claude setup-token`
// Validates prefix "sk-ant-oat01-" and minimum length (80 chars)
cred, err := auth.LoginSetupToken(os.Stdin)
```

### PKCE Support

OAuth flows use PKCE (Proof Key for Code Exchange) for enhanced security:

```go
// pkg/auth/pkce.go
type PKCE struct {
    verifier  string
    challenge string
    method    string // "S256" or "plain"
}
```

### Credential Storage

Credentials are stored in `~/.picoclaw/auth.json` with restricted permissions (0o600):

```go
// pkg/auth/store.go
func SaveStore(store *AuthStore) error {
    return fileutil.WriteFileAtomic(path, data, 0o600)
}
```

Credential structure:
```go
type AuthCredential struct {
    AccessToken  string    `json:"access_token"`
    RefreshToken string    `json:"refresh_token,omitempty"`
    AccountID    string    `json:"account_id,omitempty"`
    ExpiresAt    time.Time `json:"expires_at,omitempty"`
    Provider     string    `json:"provider"`
    AuthMethod   string    `json:"auth_method"` // "oauth", "token", "browser", "device-code"
    Email        string    `json:"email,omitempty"`
    ProjectID    string    `json:"project_id,omitempty"`
}
```

### Token Validation

```go
func (c *AuthCredential) IsExpired() bool {
    if c.ExpiresAt.IsZero() { return false }
    return time.Now().After(c.ExpiresAt)
}

func (c *AuthCredential) NeedsRefresh() bool {
    if c.ExpiresAt.IsZero() { return false }
    return time.Now().Add(5 * time.Minute).After(c.ExpiresAt)
}
```

## Security Roadmap (from ROADMAP.md)

### Input Defense and Permission Control

**Prompt Injection Defense**
- Harden JSON extraction logic to prevent LLM manipulation
- Status: Planned

**Tool Abuse Prevention**
- Strict parameter validation for generated commands
- Ensure commands stay within safe boundaries
- Status: Planned

**SSRF Protection**
- Built-in blocklists for network tools
- Prevent accessing internal IPs (LAN, Metadata services)
- Status: Planned

### Sandboxing and Isolation

**Filesystem Sandbox**
- Restrict file R/W operations to specific directories
- Status: Planned

**Context Isolation**
- Prevent data leakage between user sessions or channels
- Status: Planned

**Privacy Redaction**
- Auto-redact sensitive info (API Keys, PII) from logs
- Implemented via `SensitiveDataReplacer()` in security config
- Status: Partial (implementation exists, deployment incomplete)

### Cryptographic Upgrades

**Modern Algorithms**
- Adopt ChaCha20-Poly1305 for secret storage
- Status: Planned

**OAuth 2.0 Flow**
- Deprecate hardcoded API keys in CLI
- Move to secure OAuth flows
- Status: In Progress (browser/device code flows implemented, token paste still supported)

## Atomic File Operations

All sensitive file writes use atomic operations with explicit sync:

```go
// pkg/fileutil/file.go
func WriteFileAtomic(path string, data []byte, perm os.FileMode) error {
    // 1. Create temp file in same directory
    // 2. Write data
    // 3. Sync to physical storage (critical for SD cards)
    // 4. Set permissions
    // 5. Close
    // 6. Atomic rename
    // 7. Sync directory metadata
}
```

This prevents:
- Partial writes on crash
- Corruption during power loss
- Unauthorized read during write (0o600 immediately)

## Channel Security

Each channel has dedicated security structs in `.security.yml`:

```yaml
channels:
  telegram:
    token: "123456:ABC..."           # Bot token
  feishu:
    app_secret: "xxx"
    encrypt_key: "xxx"
    verification_token: "xxx"
  discord:
    token: "xxx"
  slack:
    bot_token: "xoxb-..."
    app_token: "xapp-..."
  matrix:
    access_token: "xxx"
```

## Known Security Limitations

1. **Pre-v1.0 Status**: Not recommended for production with sensitive data
2. **gosec Linter Disabled**: Security scanning linter is disabled in golangci.yaml due to failures
3. **Token Paste Method**: Direct API token pasting bypasses OAuth flow security
4. **No Encryption at Rest**: Auth store is JSON, not encrypted (relies on filesystem permissions)
5. **Environment Variable Exposure**: API keys in environment variables may appear in process listings

## Best Practices for Deployment

1. **Use OAuth flows** instead of direct API keys when possible
2. **Set `PICOCLAW_HOME`** to a protected directory
3. **Use filesystem permissions** to restrict access to `~/.picoclaw/`
4. **Enable channel encryption** where supported (Feishu encrypt_key)
5. **Review logs** before sharing - sensitive data filtering is partial
6. **Keep updated** - security hardening is active development
