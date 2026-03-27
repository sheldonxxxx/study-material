# Feature Deep-Dive: OpenClaw Migration Path

## Feature Name
**OpenClaw Migration Path** — Automatic migration from OpenClaw to Hermes Agent

---

## Core Implementation Files

| File | Purpose |
|------|---------|
| `hermes_cli/claw.py` (~309 lines) | CLI wrapper; handles user interaction, script discovery, and report formatting |
| `optional-skills/migration/openclaw-migration/scripts/openclaw_to_hermes.py` (~2498 lines) | Core migration engine with the `Migrator` class |
| `hermes_cli/default_soul.py` (~12 lines) | Default SOUL.md template seeded into Hermes home |
| `optional-skills/migration/openclaw-migration/SKILL.md` (~298 lines) | Agent-guided migration skill definition |
| `tests/hermes_cli/test_claw.py` (~341 lines) | Unit tests for the CLI wrapper |
| `tests/skills/test_openclaw_migration.py` | Integration tests for the migration script |

---

## How the Feature Works

### Architecture Overview

The migration is structured as a **two-layer system**:

1. **CLI Layer** (`hermes_cli/claw.py`): A thin wrapper that:
   - Discovers the migration script in either the repo (`optional-skills/`) or the installed skill location (`~/.hermes/skills/`)
   - Dynamically loads the migration module via `importlib.util`
   - Handles user confirmation prompts and report formatting
   - Provides a friendly CLI interface: `hermes claw migrate [options]`

2. **Migration Engine** (`openclaw_to_hermes.py`): A self-contained `Migrator` class that:
   - Scans `~/.openclaw/` for migration candidates
   - Transforms and copies compatible data to `~/.hermes/`
   - Produces structured reports with detailed per-item status

### Migration Flow

```
hermes claw migrate --preset full --overwrite
    |
    v
hermes_cli/claw.py (_cmd_migrate)
    |-- _find_migration_script() --> finds openclaw_to_hermes.py
    |-- _load_migration_module() --> importlib动态加载
    |-- Migrator(source_root, target_root, execute=True, ...)
    |       |
    |       v
    |   Migrator.migrate() --> runs all selected migration tasks
    |       |-- migrate_soul()
    |       |-- migrate_memory() [with dedup + char limits]
    |       |-- migrate_command_allowlist()
    |       |-- migrate_skills() [with conflict handling]
    |       |-- migrate_discord_settings() / migrate_slack_settings() / etc.
    |       |-- migrate_mcp_servers()
    |       |-- archive_docs() [unmapped items]
    |       |-- generate_migration_notes()
    |       v
    |   returns report dict
    |
    v
_print_migration_report() --> formatted console output
```

### Preset System

Two named presets control what gets migrated:

| Preset | Includes Secrets | Items |
|--------|-----------------|-------|
| `user-data` | No | soul, memory, user-profile, skills, command-allowlist, messaging settings, TTS assets, workspace agents, archived items |
| `full` | Yes (allowlisted only) | All of `user-data` + `secret-settings` (API keys) |

**Security design**: Secrets are **opt-in** even in `full` mode via `--migrate-secrets` flag. The `full` preset just enables this flag automatically. Only 6 allowlisted secrets are ever migrated:
- `TELEGRAM_BOT_TOKEN`
- `OPENROUTER_API_KEY`
- `OPENAI_API_KEY`
- `ANTHROPIC_API_KEY`
- `ELEVENLABS_API_KEY`
- `VOICE_TOOLS_OPENAI_KEY`

### What Gets Migrated

**Direct migrations** (with 1:1 mapping):
- `SOUL.md` -> `~/.hermes/SOUL.md`
- `MEMORY.md` / `USER.md` -> `~/.hermes/memories/` (with dedup + char limit enforcement)
- `workspace/skills/` -> `~/.hermes/skills/openclaw-imports/`
- `workspace/tts/` -> `~/.hermes/tts/`
- `exec-approvals.json` patterns -> merged into `config.yaml command_allowlist`
- Channel tokens/allowlists -> `.env` file
- MCP server definitions -> `config.yaml mcp_servers`
- Model config -> `config.yaml model`
- TTS config -> `config.yaml tts`
- Agent defaults -> various `config.yaml` fields

**Archived items** (no direct mapping, saved for manual review):
- `IDENTITY.md`, `TOOLS.md`, `HEARTBEAT.md`
- Plugin configs, hook configs, cron job configs
- Advanced channel configs (group settings, thread bindings)
- Memory backend configs, skills registry configs
- UI/identity settings, logging/diagnostics configs

---

## Notable Code Patterns and Solutions

### 1. Dynamic Module Loading
The migration script is loaded dynamically to allow it to be either:
- Bundled in the repo at `optional-skills/migration/openclaw-migration/scripts/`
- Installed via Skills Hub to `~/.hermes/skills/migration/openclaw-migration/scripts/`

```python
def _load_migration_module(script_path: Path):
    spec = importlib.util.spec_from_file_location("openclaw_to_hermes", script_path)
    mod = importlib.util.module_from_spec(spec)
    sys.modules[spec.name] = mod  # Required for Python 3.11+ dataclass resolution
    spec.loader.exec_module(mod)
    return mod
```

### 2. Memory Entry Deduplication and Merging
Memory entries from OpenClaw are parsed, deduplicated, and merged with existing Hermes memories with character limit enforcement:

```python
def merge_entries(existing, incoming, limit):
    merged = list(existing)
    seen = {normalize_text(entry) for entry in existing}
    for entry in incoming:
        normalized = normalize_text(entry)
        if normalized in seen:
            stats["duplicates"] += 1
            continue
        candidate_len = len(ENTRY_DELIMITER.join(merged)) + len(entry)
        if candidate_len > limit:
            stats["overflowed"] += 1
            overflowed.append(entry)  # Saved to overflow file
            continue
        merged.append(entry)
```

The `ENTRY_DELIMITER = "\n§\n"` is used to separate entries in the memory file.

### 3. Markdown Entry Extraction
Complex parsing of OpenClaw's flat markdown format into structured entries:

```python
def extract_markdown_entries(text: str) -> List[str]:
    # Parses headings, bullets, code blocks, tables
    # Builds context prefix from headings: "Context > Heading > Subheading"
    # Deduplicates based on normalized text
```

Handles code blocks (skipped), tables (skipped), headings (used for context), and bullet points (extracted as entries).

### 4. Skill Conflict Resolution
Three modes for handling skill name conflicts:
- `skip` (default): Leave existing Hermes skill, skip imported one
- `overwrite`: Replace existing with imported (with backup)
- `rename`: Import under new name (`skill-name-imported`, `skill-name-imported-2`, etc.)

```python
def resolve_skill_destination(self, destination: Path) -> Path:
    if self.skill_conflict_mode != "rename" or not destination.exists():
        return destination
    suffix = "-imported"
    candidate = destination.with_name(destination.name + suffix)
    counter = 2
    while candidate.exists():
        candidate = destination.with_name(f"{destination.name}{suffix}-{counter}")
        counter += 1
    return candidate
```

### 5. Channel Configuration Mapping
Extensive mapping of OpenClaw's channel config to Hermes environment variables:

```python
CHANNEL_ENV_MAP = {
    "matrix": {"token": "MATRIX_ACCESS_TOKEN", "allowFrom": "MATRIX_ALLOWED_USERS",
               "extras": {"homeserverUrl": "MATRIX_HOMESERVER_URL", ...}},
    "mattermost": {"token": "MATTERMOST_BOT_TOKEN", ...},
    "irc": {"extras": {"server": "IRC_SERVER", "nick": "IRC_NICK", ...}},
    # ... 10+ channels
}
```

### 6. Backup Before Overwrite
Every destructive operation creates a backup first:

```python
def maybe_backup(self, path: Path) -> Optional[Path]:
    if not self.execute or not self.backup_dir or not path.exists():
        return None
    return backup_existing(path, self.backup_dir)
```

Backups go to `~/.hermes/migration/openclaw/<timestamp>/backups/`.

### 7. Dry-Run Mode
All operations support `--dry-run` which simulates changes without modifying anything. The `execute` flag is passed through the entire system:

```python
if self.execute:
    shutil.copy2(source, destination)
    self.record(kind, source, destination, "migrated", ...)
else:
    self.record(kind, source, destination, "migrated", "Would copy")
```

### 8. Run-If-Selected Pattern
Clean separation between selection logic and execution:

```python
def run_if_selected(self, option_id: str, func) -> None:
    if self.is_selected(option_id):
        func()
        return
    meta = MIGRATION_OPTION_METADATA[option_id]
    self.record(option_id, None, None, "skipped", "Not selected for this run", ...)

# Usage:
self.run_if_selected("soul", self.migrate_soul)
self.run_if_selected("skills", self.migrate_skills)
```

---

## Technical Debt and Concerns

### 1. Large, Complex Migration Script
The `openclaw_to_hermes.py` script is ~2500 lines with a single `Migrator` class handling 35+ migration types. This is difficult to maintain and test comprehensively. Consider splitting into smaller, focused migrator classes per domain (e.g., `ConfigMigrator`, `SkillsMigrator`, `MemoryMigrator`).

### 2. Memory Overflow Can Lose Data
When `MEMORY.md` or `USER.md` exceeds character limits, overflow entries are saved to a file but are essentially forgotten:

```python
if candidate_len > limit:
    stats["overflowed"] += 1
    overflowed.append(entry)  # User must manually find and re-add these
```

There is no mechanism to alert the user post-migration that overflow occurred, nor any UI to review and re-import overflow entries.

### 3. Archive Items Are Effectively Dead
Items archived for "manual review" (plugins, hooks, cron jobs, memory backend configs, etc.) are saved to `archive/` but there is no follow-up system to actually review them. Most users will not know to check `~/.hermes/migration/openclaw/<timestamp>/archive/`.

### 4. OpenClaw Workspace Detection Heuristic
The code prefers `workspace/` over `workspace.default/`:

```python
source_candidate("workspace/SOUL.md", "workspace.default/SOUL.md")
```

This heuristic works but assumes a single primary workspace, which may not match all OpenClaw setups.

### 5. YAML Dependency for Config Migration
The migration requires PyYAML for config.yaml modifications but handles absence gracefully:

```python
try:
    import yaml
except Exception:
    yaml = None

def dump_yaml_file(path: Path, data: Dict[str, Any]) -> None:
    if yaml is None:
        raise RuntimeError("PyYAML is required to update Hermes config.yaml")
```

However, if PyYAML is missing during migration of config items (command allowlist, model config, etc.), those items will fail with an error rather than being skipped or archived.

### 6. API Key Matching by Provider Name/URL
Provider API key migration uses heuristic matching:

```python
if "openrouter" in base_url.lower():
    env_var = "OPENROUTER_API_KEY"
elif "openai.com" in base_url.lower():
    env_var = "OPENAI_API_KEY"
```

This string-matching approach is fragile if provider URLs change or if multiple providers are configured.

### 7. No Transactional Guarantees
If migration fails midway (e.g., due to disk space or permissions), there is no rollback mechanism. Partial migrations leave the Hermes home in an inconsistent state. The backup system helps but requires `--overwrite` to restore.

### 8. Skill Discovery Relies on SKILL.md
Skills are identified by the presence of a `SKILL.md` file:

```python
skill_dirs = [p for p in sorted(source_root.iterdir())
              if p.is_dir() and (p / "SKILL.md").exists()]
```

This is the same standard used by Hermes skills, but OpenClaw skills may have used a different structure.

---

## Key Code Snippets

### CLI Entry Point (hermes_cli/claw.py)
```python
def claw_command(args):
    """Route hermes claw subcommands."""
    action = getattr(args, "claw_action", None)
    if action == "migrate":
        _cmd_migrate(args)
    else:
        print("Usage: hermes claw migrate [options]")
```

### Migration Report Summary
```python
# From openclaw_to_hermes.py
report = {
    "timestamp": self.timestamp,
    "mode": "execute" if self.execute else "dry-run",
    "source_root": str(self.source_root),
    "target_root": str(self.target_root),
    "summary": {"migrated": 0, "archived": 0, "skipped": 0, "conflict": 0, "error": 0},
    "items": [asdict(item) for item in self.items],
}
```

### Secret Migration Allowlist
```python
SUPPORTED_SECRET_TARGETS={
    "TELEGRAM_BOT_TOKEN",
    "OPENROUTER_API_KEY",
    "OPENAI_API_KEY",
    "ANTHROPIC_API_KEY",
    "ELEVENLABS_API_KEY",
    "VOICE_TOOLS_OPENAI_KEY",
}
```

### Command Allowlist Merge
```python
config = load_yaml_file(destination)
current = config.get("command_allowlist", [])
merged = sorted(dict.fromkeys(list(current) + patterns))
config["command_allowlist"] = merged
dump_yaml_file(destination, config)
```
