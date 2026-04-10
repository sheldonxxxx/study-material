# Feature Deep Dive - Batch 4 (Features 10-12)

## Feature 10: Agent Social Network

### Description (from README)
Connect to agent social networks (Moltbook, ClawdChat) via skill installation.

### Implementation Analysis

**Key Files:**
- `nanobot/skills/clawhub/SKILL.md` - ClawHub skill marketplace integration
- `nanobot/agent/skills.py` - Skills loader with workspace skills support

### How It Actually Works

The "Agent Social Network" feature is not a native protocol or special integration. Instead, it works through a clever mechanism: the agent simply reads skill.md files from URLs that teach it how to join these networks.

**The Flow:**
1. User sends the message: `Read https://moltbook.com/skill.md and follow the instructions to join Moltbook`
2. The agent uses its `web_fetch` tool to read the URL
3. The skill.md file contains instructions on how to install/configure the agent to join
4. The agent follows those instructions (which typically involve installing a skill via ClawHub)

**Evidence from README (line 827-834):**
```markdown
🐈 nanobot is capable of linking to the agent social network (agent community).
**Just send one message and your nanobot joins automatically!**

| Platform | How to Join (send this message to your bot) |
|----------|-------------|
| [**Moltbook**](https://www.moltbook.com/) | `Read https://moltbook.com/skill.md and follow the instructions to join Moltbook` |
| [**ClawdChat**](https://clawdchat.ai/) | `Read https://clawdchat.ai/skill.md and follow the instructions to join ClawdChat` |
```

### ClawHub Skill Marketplace

The actual skill marketplace is **ClawHub** (`https://clawhub.ai`), which has a CLI tool (`npx clawhub`) for searching and installing skills.

**SKILL.md (`nanobot/skills/clawhub/SKILL.md`):**
```markdown
---
name: clawhub
description: Search and install agent skills from ClawHub, the public skill registry.
homepage: https://clawhub.ai
metadata: {"nanobot":{"emoji":"🦞"}}
---

# ClawHub

Public skill registry for AI agents. Search by natural language (vector search).

## When to use

Use this skill when the user asks any of:
- "find a skill for …"
- "search for skills"
- "install a skill"
- "what skills are available?"
- "update my skills"

## Search

```bash
npx --yes clawhub@latest search "web scraping" --limit 5
```

## Install

```bash
npx --yes clawhub@latest install <slug> --workdir ~/.nanobot/workspace
```

Replace `<slug>` with the skill name from search results. This places the skill into `~/.nanobot/workspace/skills/`, where nanobot loads workspace skills from. Always include `--workdir`.
```

### Skills Loading Architecture

Skills are loaded from two locations (from `nanobot/agent/skills.py` lines 36-52):

```python
# Workspace skills (highest priority)
if self.workspace_skills.exists():
    for skill_dir in self.workspace_skills.iterdir():
        if skill_dir.is_dir():
            skill_file = skill_dir / "SKILL.md"
            if skill_file.exists():
                skills.append({"name": skill_dir.name, "path": str(skill_file), "source": "workspace"})

# Built-in skills
if self.builtin_skills and self.builtin_skills.exists():
    for skill_dir in self.builtin_skills.iterdir():
        if skill_dir.is_dir():
            skill_file = skill_dir / "SKILL.md"
            if skill_file.exists() and not any(s["name"] == skill_dir.name for s in skills):
                skills.append({"name": skill_dir.name, "path": str(skill_file), "source": "builtin"})
```

**Key insight:** Workspace skills (`~/.nanobot/workspace/skills/`) take priority over built-in skills. This allows users to override bundled skills with custom versions.

### Notable Patterns and Technical Debt

1. **No native protocol**: The "social network" is just a convention - URLs pointing to SKILL.md files that teach the agent how to join. There's no special API or integration.

2. **Skill installation is external**: ClawHub skills are installed via `npx clawhub@latest` - Node.js CLI that runs outside Python. This is a shell tool, not a native Python integration.

3. **Works via agent's existing tools**: The agent uses its `web_fetch` tool to read skill.md files, and `shell` tool to run npx commands. No special social network code.

4. **No verification**: The system doesn't verify if a skill was successfully installed or if the join process completed. Success is assumed if no error is raised.

### Error Handling and Edge Cases

- If `npx` is not installed, ClawHub skill installation fails (but this is not caught specially)
- If the URL returns non-markdown content, the agent may fail to parse instructions
- Workspace skills directory must exist (`~/.nanobot/workspace/skills/`) - created during onboard

---

## Feature 11: Multi-Instance Support

### Description (from README)
Run multiple nanobot instances with separate configs, workspaces, and ports simultaneously.

### Implementation Analysis

**Key Files:**
- `nanobot/config/loader.py` - Config path management with global state
- `nanobot/config/paths.py` - Runtime path derivation from config path
- `nanobot/config/schema.py` - Pydantic configuration schemas
- `nanobot/cli/commands.py` - CLI with `--config` and `--workspace` flags

### How It Actually Works

Multi-instance support relies on two mechanisms:
1. **Custom config file path** via `--config` flag
2. **Custom workspace directory** via `--workspace` flag

**Global config path state** (`nanobot/config/loader.py` lines 11-25):
```python
# Global variable to store current config path (for multi-instance support)
_current_config_path: Path | None = None

def set_config_path(path: Path) -> None:
    """Set the current config path (used to derive data directory)."""
    global _current_config_path
    _current_config_path = path

def get_config_path() -> Path:
    """Get the configuration file path."""
    if _current_config_path:
        return _current_config_path
    return Path.home() / ".nanobot" / "config.json"
```

**Runtime paths derived from config location** (`nanobot/config/paths.py`):
```python
def get_data_dir() -> Path:
    """Return the instance-level runtime data directory."""
    return ensure_dir(get_config_path().parent)

def get_runtime_subdir(name: str) -> Path:
    """Return a named runtime subdirectory under the instance data dir."""
    return ensure_dir(get_data_dir() / name)

def get_cron_dir() -> Path:
    """Return the cron storage directory."""
    return get_runtime_subdir("cron")

def get_logs_dir() -> Path:
    """Return the logs directory."""
    return get_runtime_subdir("logs")

def get_media_dir(channel: str | None = None) -> Path:
    """Return the media directory, optionally namespaced per channel."""
    base = get_runtime_subdir("media")
    return ensure_dir(base / channel) if channel else base
```

### CLI Support for Multi-Instance

The `--config` and `--workspace` flags are available on multiple commands:

**`gateway` command (`cli/commands.py` lines 499-505):**
```python
@app.command()
def gateway(
    port: int | None = typer.Option(None, "--port", "-p", help="Gateway port"),
    workspace: str | None = typer.Option(None, "--workspace", "-w", help="Workspace directory"),
    verbose: bool = typer.Option(False, "--verbose", "-v", help="Verbose output"),
    config: str | None = typer.Option(None, "--config", "-c", help="Path to config file"),
):
```

**`agent` command (`cli/commands.py` lines 708-716):**
```python
@app.command()
def agent(
    message: str = typer.Option(None, "--message", "-m", help="Message to send to the agent"),
    session_id: str = typer.Option("cli:direct", "--session", "-s", help="Session ID"),
    workspace: str | None = typer.Option(None, "--workspace", "-w", help="Workspace directory"),
    config: str | None = typer.Option(None, "--config", "-c", help="Config file path"),
    ...
):
```

**`onboard` command (`cli/commands.py` lines 250-255):**
```python
@app.command()
def onboard(
    workspace: str | None = typer.Option(None, "--workspace", "-w", help="Workspace directory"),
    config: str | None = typer.Option(None, "--config", "-c", help="Path to config file"),
    wizard: bool = typer.Option(False, "--wizard", help="Use interactive wizard"),
):
```

### Config Loading with Override

The `_load_runtime_config()` function (`cli/commands.py` lines 445-462):
```python
def _load_runtime_config(config: str | None = None, workspace: str | None = None) -> Config:
    """Load config and optionally override the active workspace."""
    from nanobot.config.loader import load_config, set_config_path

    config_path = None
    if config:
        config_path = Path(config).expanduser().resolve()
        if not config_path.exists():
            console.print(f"[red]Error: Config file not found: {config_path}[/red]")
            raise typer.Exit(1)
        set_config_path(config_path)
        console.print(f"[dim]Using config: {config_path}[/dim]")

    loaded = load_config(config_path)
    _warn_deprecated_config_keys(config_path)
    if workspace:
        loaded.agents.defaults.workspace = workspace
    return loaded
```

### What Gets Isolated Per Instance

Based on code analysis:

| Component | Isolation Method |
|-----------|-----------------|
| Config file | `--config` flag sets `set_config_path()` |
| Workspace | `--workspace` flag overrides `agents.defaults.workspace` |
| Sessions | Stored in workspace under `~/.nanobot/workspace/sessions/` |
| Memory | Stored in workspace under `~/.nanobot/workspace/memory/` |
| Cron jobs | Stored in workspace under `~/.nanobot/workspace/cron/` |
| Skills | Workspace skills in `~/.nanobot/workspace/skills/` |
| Port | `--port` flag on gateway (default 18790) |
| Channels | Each channel reads from the shared config |

### Test Coverage

The test file `tests/config/test_config_paths.py` verifies runtime directories follow the config path:

```python
def test_runtime_dirs_follow_config_path(monkeypatch, tmp_path: Path) -> None:
    config_file = tmp_path / "instance-a" / "config.json"
    monkeypatch.setattr("nanobot.config.paths.get_config_path", lambda: config_file)

    assert get_data_dir() == config_file.parent
    assert get_runtime_subdir("cron") == config_file.parent / "cron"
    assert get_cron_dir() == config_file.parent / "cron"
    assert get_logs_dir() == config_file.parent / "logs"
```

### Notable Patterns and Technical Debt

1. **Global mutable state**: The `_current_config_path` global variable is not thread-safe. If multiple instances run in the same process, they could interfere.

2. **Shared CLI history**: `get_cli_history_path()` always returns `~/.nanobot/history/cli_history` regardless of which instance is running - this is intentional (single CLI history) but could be confusing.

3. **Shared bridge installation**: WhatsApp bridge is installed to `~/.nanobot/bridge` globally, not per-instance.

4. **Port conflicts**: Running multiple gateway instances on the same machine requires different `--port` values. No automatic port discovery or shifting.

5. **No instance registry**: There's no command to list running instances or their status.

### Error Handling and Edge Cases

- If config file doesn't exist when using `--config`: Error message and exit
- If workspace doesn't exist: Created automatically during onboard
- If port is already in use: Connection refused error from the network stack
- If two instances use same workspace: They would share session/memory/cron - no locking or warning

---

## Feature 12: Docker/Service

### Description (from README)
Docker deployment via docker-compose or standalone Dockerfile, plus systemd user service for auto-start.

### Implementation Analysis

**Key Files:**
- `Dockerfile` - Container definition
- `docker-compose.yml` - Multi-container orchestration
- `README.md` lines 1565-1656 - Documentation

### Dockerfile Analysis

**Base image and setup:**
```dockerfile
FROM ghcr.io/astral-sh/uv:python3.12-bookworm-slim

# Install Node.js 20 for the WhatsApp bridge
RUN apt-get update && \
    apt-get install -y --no-install-recommends curl ca-certificates gnupg git openssh-client && \
    mkdir -p /etc/apt/keyrings && \
    curl -fsSL https://deb.nodesource.com/gpgkey/nodesource-repo.gpg.key | gpg --dearmor -o /etc/apt/keyrings/nodesource.gpg && \
    echo "deb [signed-by=/etc/apt/keyrings/nodesource.gpg] https://deb.nodesource.com/node_20.x nodistro main" > /etc/apt/sources.list.d/nodesource.list && \
    apt-get update && \
    apt-get install -y --no-install-recommends nodejs && \
    apt-get purge -y gnupg && \
    apt-get autoremove -y && \
    rm -rf /var/lib/apt/lists/*
```

**Key observations:**
1. Uses `uv` (Astral's fast Python package manager) instead of pip
2. Installs Node.js 20 specifically for the WhatsApp bridge
3. Git is installed with URL rewriting to use HTTPS instead of SSH

**Build and installation:**
```dockerfile
WORKDIR /app

# Install Python dependencies first (cached layer)
COPY pyproject.toml README.md LICENSE ./
RUN mkdir -p nanobot bridge && touch nanobot/__init__.py && \
    uv pip install --system --no-cache . && \
    rm -rf nanobot bridge

# Copy the full source and install
COPY nanobot/ nanobot/
COPY bridge/ bridge/
RUN uv pip install --system --no-cache .

# Build the WhatsApp bridge
RUN git config --global url."https://github.com/".insteadOf "ssh://git@github.com/"

WORKDIR /app/bridge
RUN npm install && npm run build
WORKDIR /app

# Create config directory
RUN mkdir -p /root/.nanobot

# Gateway default port
EXPOSE 18790

ENTRYPOINT ["nanobot"]
CMD ["status"]
```

### Docker Compose Configuration

```yaml
x-common-config: &common-config
  build:
    context: .
    dockerfile: Dockerfile
  volumes:
    - ~/.nanobot:/root/.nanobot

services:
  nanobot-gateway:
    container_name: nanobot-gateway
    <<: *common-config
    command: ["gateway"]
    restart: unless-stopped
    ports:
      - 18790:18790
    deploy:
      resources:
        limits:
          cpus: '1'
          memory: 1G
        reservations:
          cpus: '0.25'
          memory: 256M

  nanobot-cli:
    <<: *common-config
    profiles:
      - cli
    command: ["status"]
    stdin_open: true
    tty: true
```

**Key observations:**
1. Uses `profiles` to separate CLI and gateway services - CLI only runs with `docker compose --profile cli`
2. Volume mount `~/.nanobot:/root/.nanobot` persists config and workspace
3. Resource limits set (1 CPU, 1GB RAM max)
4. `restart: unless-stopped` ensures gateway restarts on failure or reboot

### Systemd User Service Documentation

The README documents (not code) a systemd user service setup:

```ini
[Unit]
Description=Nanobot Gateway
After=network.target

[Service]
Type=simple
ExecStart=%h/.local/bin/nanobot gateway
Restart=always
RestartSec=10
NoNewPrivileges=yes
ProtectSystem=strict
ReadWritePaths=%h

[Install]
WantedBy=default.target
```

**Important note:** These service files are NOT in the repository. The README instructs users to create them manually.

### What Gets Persisted

The volume mount `~/.nanobot:/root/.nanobot` includes:
- Config: `~/.nanobot/config.json`
- Workspace: `~/.nanobot/workspace/` (sessions, memory, cron jobs, skills)
- CLI history: `~/.nanobot/history/`
- WhatsApp auth: `~/.nanobot/bridge/` (if used)

### Notable Patterns and Technical Debt

1. **WhatsApp bridge built inside container**: The TypeScript bridge is compiled during Docker image build, not at runtime. This makes the image larger but self-contained.

2. **Git URL rewriting**: `git config --global url."https://github.com/".insteadOf "ssh://git@github.com/"` allows fetching Git dependencies without SSH keys. This is a workaround for GitHub's SSH fingerprint changes in Docker environments.

3. **No multi-platform build**: The Dockerfile is AMD64-only (no ARM support for Apple Silicon).

4. **Build caching strategy**: The `uv pip install` is run twice - once with just pyproject.toml (for caching dependencies), then again with full source. This is an optimization for CI/CD.

5. **Systemd files not in repo**: The README instructs users to manually create systemd service files. No `nanobot-gateway.service` is provided in the repository.

6. **No healthcheck**: Docker container has no `HEALTHCHECK` instruction. Docker doesn't know if the gateway is healthy.

7. **npm in container**: Node.js/npm is installed in the container for WhatsApp bridge. This adds ~100MB to the image.

8. **No dockerignore**: Repository has no `.dockerignore` file - all files are sent to Docker context.

### CI/CD

The GitHub Actions workflow (`ci.yml`) runs tests on Python 3.11, 3.12, 3.13 but does NOT build or push Docker images.

```yaml
- name: Run tests
  run: uv run pytest tests/
```

**No Docker image publishing workflow exists** in this repository.

### Error Handling and Edge Cases

- If `npm install` fails in Docker build: Build fails with error message
- If WhatsApp bridge fails to build: Docker build fails
- If port 18790 already in use on host: Docker port mapping fails
- If config file is missing: `nanobot onboard` must be run inside container first

---

## Code vs Documentation Discrepancies

### Feature 10 (Agent Social Network)
- **Documentation claims**: "Just send one message and your nanobot joins automatically!"
- **Reality**: The agent must successfully:
  1. Fetch the skill.md URL
  2. Parse the markdown instructions
  3. Execute the installation commands (typically via shell)
  4. Start a new session to load the skill
- **Discrepancy**: "Automatically" implies no user intervention, but the agent may fail at any step and there's no recovery mechanism

### Feature 11 (Multi-Instance Support)
- **Documentation claims**: "Run multiple nanobot instances with separate configs, workspaces, and ports simultaneously"
- **Reality**: Partially true - configs and workspaces are isolated, but:
  - Global state (`_current_config_path`) is not thread-safe
  - CLI history is shared across instances
  - WhatsApp bridge is shared
- **Discrepancy**: Minor - the core isolation works as documented

### Feature 12 (Docker/Service)
- **Documentation claims**: "Docker deployment via docker-compose or standalone Dockerfile, plus systemd user service"
- **Reality**: Docker is fully implemented and works. Systemd is documented but NO service files exist in the repository.
- **Discrepancy**: Users must manually create systemd service files from README instructions. No `nanobot.service` or `nanobot-gateway.service` is provided.

---

## Summary

| Feature | Implementation Quality | Notes |
|---------|------------------------|-------|
| Agent Social Network | Basic | Works via agent reading skill.md files and executing shell commands. No native protocol. |
| Multi-Instance Support | Functional but fragile | Uses global mutable state. Works for basic use cases but not thread-safe. |
| Docker/Service | Well-implemented | Docker works correctly. Systemd only documented, not provided as code. |
