# Security: DeerFlow 2.0

## Security Model Overview

DeerFlow is designed for **local trusted environments** by default. Given its capabilities — system command execution, file operations, and business logic invocation — the primary security risk is unauthorized access in untrusted deployments.

**Critical assumption**: The agent runs in an environment where all tool access is intentional and authorized by the operator.

## Security Architecture

### Defense Layers

```
┌─────────────────────────────────────────────────────────────────────┐
│                        DeerFlow Security Layers                       │
│                                                                      │
│  Layer 1: Deployment Isolation                                         │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Local trusted network (127.0.0.1 only by default)             │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                        │
│  Layer 2: Sandbox Isolation                                           │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Docker containers / Kubernetes pods                          │   │
│  │ Virtual path translation (/mnt/user-data/*)                  │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                        │
│  Layer 3: Guardrails Authorization                                   │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Pre-tool-call policy evaluation                              │   │
│  │ Pluggable providers (Allowlist, OAP, Custom)                 │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                        │
│  Layer 4: Tool Execution                                             │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Path translation, command blocking, error handling            │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

## Sandbox Security

### Three Isolation Modes

| Mode | Isolation | Use Case |
|------|-----------|----------|
| **Local** | None (host execution) | Development only |
| **Docker** | Container isolation | Production single-node |
| **Kubernetes** | Pod isolation via provisioner | Production multi-node |

### Virtual Path System

Agents operate in virtual paths that map to thread-isolated physical directories:

```
Agent View:                     Physical View:
/mnt/user-data/workspace   ──►  backend/.deer-flow/threads/{thread_id}/user-data/workspace
/mnt/user-data/uploads     ──►  backend/.deer-flow/threads/{thread_id}/user-data/uploads
/mnt/user-data/outputs     ──►  backend/.deer-flow/threads/{thread_id}/user-data/outputs
/mnt/skills/public         ──►  deer-flow/skills/public
/mnt/acp-workspace         ──►  backend/.deer-flow/threads/{thread_id}/acp-workspace
```

### Path Translation

`replace_virtual_path()` and `replace_virtual_paths_in_command()` translate paths before execution. The `is_local_sandbox()` check determines if `sandbox_id == "local"` to skip translation.

## Guardrails System

### Purpose

Guardrails provide **deterministic, policy-driven authorization** for tool calls — independent of sandboxing (which provides process isolation, not semantic authorization).

### Architecture

```
Tool Call ──► GuardrailMiddleware ──► GuardrailProvider.evaluate()
                                          │
                    ┌────────────────────┼────────────────────┐
                    │                    │                    │
              AllowlistProvider     OAPProvider         Custom Provider
              (built-in, zero dep)   (open standard)     (user-defined)
```

### Three Provider Options

**1. AllowlistProvider (Zero Dependencies)**

```yaml
guardrails:
  enabled: true
  provider:
    use: deerflow.guardrails.builtin:AllowlistProvider
    config:
      denied_tools: ["bash", "write_file"]
```

**2. OAP Passport Provider (Open Standard)**

Based on [Open Agent Passport](https://github.com/aporthq/aport-spec). OAP passport is a JSON document declaring agent identity, capabilities, and operational limits.

```yaml
guardrails:
  enabled: true
  provider:
    use: aport_guardrails.providers.generic:OAPGuardrailProvider
```

**3. Custom Provider**

Any class with `evaluate(request)` and `aevaluate(request)` methods:

```python
class MyGuardrailProvider:
    name = "my-company"

    def evaluate(self, request):
        if request.tool_name == "bash" and "delete" in str(request.tool_input):
            return GuardrailDecision(
                allow=False,
                reasons=[GuardrailReason(code="custom.blocked", message="delete not allowed")]
            )
        return GuardrailDecision(allow=True, reasons=[GuardrailReason(code="oap.allowed")])
```

### Guardrail Middleware Chain

Middleware executes in strict order (from `backend/CLAUDE.md`):

```
1. ThreadDataMiddleware      ── per-thread directories
2. UploadsMiddleware         ─── file upload tracking
3. SandboxMiddleware         ─── sandbox acquisition
4. DanglingToolCallMiddleware ── fix incomplete tool calls
5. GuardrailMiddleware ◄──── EVALUATES EVERY TOOL CALL
6. ToolErrorHandlingMiddleware ── exception handling
7-12. (Summarization, Title, Memory, Vision, Subagent, Clarify)
```

### Fail-Closed Default

If `fail_closed: true` (default), guardrail provider errors block tool execution:

```yaml
guardrails:
  fail_closed: true  # Default
```

## Secrets Management

### Environment Variables

API keys stored in `.env` (gitignored) or environment:

```bash
# .env (gitignored)
OPENAI_API_KEY=sk-...
TAVILY_API_KEY=...
FEISHU_APP_SECRET=...
SLACK_BOT_TOKEN=xoxb-...
TELEGRAM_BOT_TOKEN=...
```

### Config Substitution

`config.yaml` supports environment variable substitution:

```yaml
models:
  - name: gpt-4
    api_key: $OPENAI_API_KEY    # Resolved at runtime
```

### Credential Loading

`tests/test_credential_loader.py` validates credential loading patterns for CLI-backed providers (Claude Code, Codex).

## IM Channel Security

### Token-Based Authentication

| Channel | Auth Method |
|---------|-------------|
| **Telegram** | Bot API token (`TELEGRAM_BOT_TOKEN`) |
| **Slack** | Bot token (`SLACK_BOT_TOKEN`) + App token (`SLACK_APP_TOKEN`) |
| **Feishu** | App ID + App Secret (`FEISHU_APP_ID`, `FEISHU_APP_SECRET`) |

### macOS Keychain

On macOS, Claude Code auth does not probe Keychain automatically. Export explicitly:

```bash
eval "$(python3 scripts/export_claude_code_oauth.py --print-export)"
```

## Artifact Serving

### XSS Prevention

Gateway artifact serving forces active content types to download as attachments:

```python
# From backend/CLAUDE.md:
# GET /api/threads/{id}/artifacts/{path}
# Active content types (text/html, application/xhtml+xml, image/svg+xml)
# are always forced as download attachments to reduce XSS risk
```

## MCP Security

### OAuth Support

MCP servers supporting HTTP/SSE transports can use OAuth flows:
- `client_credentials`
- `refresh_token`

Tokens are automatically refreshed and injected into Authorization headers.

### Credential Handling

MCP tool definitions are cached with mtime invalidation — config changes trigger reload without restart.

## Deployment Security Recommendations

### From README.md

> **We strongly recommend deploying DeerFlow in a local trusted network environment.**

If deploying beyond localhost:

| Recommendation | Implementation |
|----------------|----------------|
| **IP Allowlist** | `iptables`, firewall ACLs |
| **Authentication Gateway** | nginx with pre-authentication |
| **Network Isolation** | Dedicated VLAN |

### Untrusted Deployment Risks

- Unauthorized bulk requests triggering high-risk operations
- Legal/compliance risks from illegal activity
- Data exfiltration via agent capabilities

## Security Testing

### Guardrail Tests

`tests/test_guardrail_middleware.py` has 25 tests covering:
- AllowlistProvider: allow, deny, combined allowlist+denylist
- GuardrailMiddleware: passthrough, denial, fail-closed, fail-open
- GraphBubbleUp: LangGraph control signals propagate through
- Async paths: `awrap_tool_call`

### Artifact Serving Tests

`test_artifacts_router.py` validates:
- Content type handling
- Download forcing for active content types

## Security Documentation

| Document | Content |
|----------|---------|
| `SECURITY.md` | Reporting vulnerabilities, supported versions |
| `backend/docs/GUARDRAILS.md` | Guardrail architecture, provider implementation |
| `README.md` | Deployment security notices |

## Known Security Considerations

1. **Default localhost binding**: Nginx and services bind to localhost by default
2. **No built-in authentication**: Relies on network-level access control
3. **Sandbox does not prevent data exfiltration**: Docker isolation does not block network requests from within sandbox
4. **Guardrails require explicit enablement**: Must be configured in `config.yaml`
5. **IM channels use long-polling/WebSocket**: Token secrets stored in environment

## Vulnerability Reporting

Per `SECURITY.md`:
> Please go to https://github.com/bytedance/deer-flow/security to report the vulnerability you find.

## Security Summary

| Aspect | Status |
|--------|--------|
| **Sandbox Isolation** | Docker/K8s available, local is unisolated |
| **Tool Authorization** | Guardrails (pluggable, opt-in) |
| **Secrets** | Environment variables only |
| **IM Auth** | Token-based per platform |
| **XSS Prevention** | Artifact download forcing |
| **Dependency Vulnerability Scanning** | Not explicitly documented |
| **Security Audits** | Not publicly documented |
| **SBOM** | Not generated |
