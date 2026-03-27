# Dependencies Analysis

## Python Dependencies

### Core AI/ML

| Dependency | Version | Purpose | Risk |
|------------|---------|---------|------|
| anthropic | >=0.45.0,<1.0.0 | Claude API | Low |
| openai | >=2.8.0 | OpenAI API | Low |
| tiktoken | >=0.12.0,<1.0.0 | Tokenizer | Low |

### Data Validation

| Dependency | Version | Purpose | Risk |
|------------|---------|---------|------|
| pydantic | >=2.12.0,<3.0.0 | Type validation | Low |
| pydantic-settings | >=2.12.0,<3.0.0 | Settings | Low |

### CLI/Terminal

| Dependency | Version | Purpose | Risk |
|------------|---------|---------|------|
| typer | >=0.20.0,<1.0.0 | CLI framework | Low |
| rich | >=14.0.0,<15.0.0 | Terminal formatting | Low |
| prompt-toolkit | >=3.0.50,<4.0.0 | Terminal UI | Low |
| questionary | >=2.0.0,<3.0.0 | Interactive prompts | Low |

### Network

| Dependency | Version | Purpose | Risk |
|------------|---------|---------|------|
| httpx | >=0.28.0,<1.0.0 | HTTP client | Medium - redirect following |
| websockets | >=16.0,<17.0 | WebSocket | Low |
| websocket-client | >=1.9.0,<2.0.0 | WebSocket client | Low |
| python-socketio | >=5.16.0,<6.0.0 | Socket.IO | Low |

### Channel SDKs

| Dependency | Version | Platform | Risk |
|------------|---------|----------|------|
| python-telegram-bot | >=22.6,<23.0 | Telegram | Low |
| slack-sdk | >=3.39.0,<4.0.0 | Slack | Low |
| lark-oapi | >=1.5.0,<2.0.0 | Feishu | Low |
| dingtalk-stream | >=0.24.0,<1.0.0 | DingTalk | Low |
| qq-botpy | >=1.2.0,<2.0.0 | QQ | Low |
| matrix-nio | >=0.25.2 | Matrix | Low |
| wecom-aibot-sdk-python | >=0.1.5 | WeCom | Low |

### Utilities

| Dependency | Version | Purpose | Risk |
|------------|---------|---------|------|
| loguru | >=0.7.3,<1.0.0 | Logging | Low |
| croniter | >=6.0.0,<7.0.0 | Cron parsing | Low |
| msgpack | >=1.1.0,<2.0.0 | Serialization | Low |
| json-repair | >=0.57.0,<1.0.0 | JSON repair | Low |
| mcp | >=1.26.0,<2.0.0 | MCP protocol | Medium |

### Development

| Dependency | Version | Purpose | Risk |
|------------|---------|---------|------|
| pytest | >=9.0.0,<10.0.0 | Testing | Low |
| pytest-asyncio | >=1.3.0,<2.0.0 | Async tests | Low |
| ruff | >=0.1.0 | Linting | Low |

---

## Node.js Dependencies (WhatsApp Bridge)

| Package | Version | Purpose | Risk |
|---------|---------|---------|------|
| @whiskeysockets/baileys | 7.0.0-rc.9 | WhatsApp Web | Medium - pre-release |
| ws | ^8.17.1 | WebSocket | Low - updated for DoS fix |
| qrcode-terminal | ^0.12.0 | QR display | Low |
| pino | ^9.0.0 | Logging | Low |

**Security note:** `ws` updated to `>=8.17.1` to fix DoS vulnerability.

---

## Dependency Patterns

### Version Pinning Strategy

- **Core dependencies:** Tight constraints (e.g., `anthropic: >=0.45.0,<1.0.0`)
- **SDK dependencies:** Loose constraints (e.g., `openai: >=2.8.0`)
- **Security-sensitive:** Explicit version (e.g., `ws: ^8.17.1`)

### Optional Dependencies

```toml
# Matrix extras
matrix-nio[e2e] >=0.25.2

# WeChat optional
qrcode[pil] >=8.0
pycryptodome >=3.20.0

# LangSmith observability
langsmith >=0.1.0
```

---

## Dependency Security Observations

1. **No pip-audit in CI** - SECURITY.md recommends running `pip-audit && npm audit` but CI doesn't execute these
2. **Pre-release WhatsApp library** - Using `7.0.0-rc.9` of Baileys
3. **SOCKS proxy support** - `python-socks` adds complexity for proxy scenarios
4. **Multiple HTTP clients** - Both `httpx` and `websocket-client` handle HTTP differently

---

## Build System

| Aspect | Details |
|--------|---------|
| Build Backend | hatchling |
| Package Manager | uv |
| Python Versions | 3.11, 3.12, 3.13 |
| Force-include | `bridge/` bundled into wheel |

```toml
[tool.hatch.build.targets.wheel.force-include]
"bridge" = "nanobot/bridge"
```

This ensures the TypeScript WhatsApp bridge is included in the Python package.
