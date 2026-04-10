# Feature Deep Dive - Batch 4: Local Model Support, Desktop Application, CLI

## Feature 1: Local Model Support

### Implementation Overview

CoPaw's local model support is a sophisticated system enabling fully offline AI inference through three distinct backends: **llama.cpp** (cross-platform), **MLX** (Apple Silicon), and **Ollama** (daemon-based). The architecture cleanly separates concerns between model management, inference backends, and the chat interface.

### Key Files and Architecture

```
src/copaw/local_models/
├── manager.py        # Singleton factory for model instances + manifest management
├── chat_model.py      # LocalChatModel (ChatModelBase wrapper for backends)
├── factory.py         # Creates LocalChatModel instances with singleton backend
├── schema.py          # Pydantic models: BackendType, LocalModelInfo, LocalModelsManifest
├── tag_parser.py      # Parses <think> and <tool_call> tags from raw model output
└── backends/
    ├── base.py        # Abstract LocalBackend interface
    ├── llamacpp_backend.py  # llama-cpp-python implementation
    └── mlx_backend.py       # mlx-lm implementation

src/copaw/providers/
├── ollama_provider.py      # OllamaProvider (delegates to Ollama daemon)
└── ollama_manager.py       # OllamaModelManager (pull/list/delete via SDK)
```

### Singleton Backend Pattern

The most notable architectural choice is the **singleton backend factory** in `factory.py`:

```python
# factory.py lines 22-29
_lock = threading.Lock()
_active_backend: Optional[LocalBackend] = None
_active_model_id: Optional[str] = None

def get_active_local_model() -> Optional[tuple[Optional[str], LocalBackend]]:
    """Return (model_id, backend) if a local model is currently loaded."""
    with _lock:
        if _active_backend is not None and _active_backend.is_loaded:
            return _active_model_id, _active_backend
        return None
```

When `create_local_chat_model()` is called with the same model_id, it **reuses the already-loaded backend** rather than unloading and reloading. This is a deliberate optimization since loading large models is expensive.

### Backend Abstraction (LocalBackend Interface)

Each backend implements the same interface:

```python
# backends/base.py lines 29-63
class LocalBackend(ABC):
    @abstractmethod
    def chat_completion(...) -> dict:  # Non-streaming, returns OpenAI-compatible dict
    @abstractmethod
    def chat_completion_stream(...) -> Iterator[dict]:  # Yields OpenAI-compatible chunks
    @abstractmethod
    def unload() -> None:  # Release VRAM/RAM
    @property
    @abstractmethod
    def is_loaded(self) -> bool:
```

### llama.cpp Backend (LlamaCppBackend)

Uses `llama-cpp-python` with configurable context size and GPU layers:

```python
# llamacpp_backend.py lines 48-82
def __init__(
    self,
    model_path: str,
    n_ctx: int = 32768,      # 32K context window default
    n_gpu_layers: int = -1,  # -1 = all layers to GPU
    verbose: bool = False,
    chat_format: Optional[str] = None,
    **kwargs,
) -> None:
    from llama_cpp import Llama
    self._llm = Llama(model_path=model_path, n_ctx=n_ctx, ...)
```

Notable: Includes a `_normalize_messages()` helper that flattens content blocks (multi-modal format) and handles `tool_calls: None` which crashes Jinja templates:

```python
# llamacpp_backend.py lines 16-42
def _normalize_messages(messages: list[dict]) -> list[dict]:
    """Normalise messages for llama-cpp-python's Jinja chat templates."""
    out: list[dict] = []
    for msg in messages:
        content = msg.get("content")
        if isinstance(content, list):
            # Flatten content blocks to plain text
            parts = []
            for block in content:
                if isinstance(block, dict):
                    parts.append(block.get("text", ""))
                elif isinstance(block, str):
                    parts.append(block)
            msg = {**msg, "content": "\n".join(parts)}
        elif content is None:
            msg = {**msg, "content": ""}
        # Jinja templates crash when iterating tool_calls that are None
        if "tool_calls" in msg and not msg["tool_calls"]:
            msg = {k: v for k, v in msg.items() if k != "tool_calls"}
        out.append(msg)
    return out
```

### MLX Backend (MlxBackend)

Apple Silicon optimization using `mlx-lm`:

```python
# mlx_backend.py lines 57-82
def __init__(self, model_path: str, max_tokens: int = 2048, **kwargs) -> None:
    import mlx_lm
    model_dir = _resolve_model_dir(model_path)  # Handle file vs directory
    self._model, self._tokenizer = mlx_lm.load(model_dir)
```

Key differences from llama.cpp:
- **Directory-based models**: MLX models are directories with safetensors + config, not single files
- **Sampler kwargs mapping**: Converts `temperature` to `temp` (MLX naming convention)
- **No native structured output**: Appends JSON schema instruction to prompt instead of using response_format

### Tag Parser for Local Model Outputs

Local models like Qwen3 embed structured data in raw text. The `tag_parser.py` handles:

1. **`<think>...</think>` blocks** (reasoning/thinking content)
2. **`<tool_call>...</tool_call>` blocks** (function calling)

Both streaming and non-streaming scenarios are handled with `has_open_tag` flags for incomplete tags during streaming.

### Ollama Integration (Separate from Local Models)

Ollama is treated differently - it manages its own models via daemon, not file-based:

```python
# ollama_manager.py lines 96-110
@staticmethod
def list_models(host: Optional[str] = None) -> List[OllamaModelInfo]:
    raw = OllamaModelManager._make_client(host).list()
    models: List[OllamaModelInfo] = []
    for m in raw.get("models", []):
        models.append(OllamaModelInfo(
            name=m.get("model", ""),
            size=m.get("size", 0) or 0,
            digest=m.get("digest"),
            modified_at=m.get("modified_at"),
        ))
    return models
```

The `OllamaProvider` uses OpenAI-compatible API (`/v1` endpoints) via `OpenAIChatModelCompat`:

```python
# ollama_provider.py lines 194-209
def get_chat_model_instance(self, model_id: str) -> ChatModelBase:
    from .openai_chat_model_compat import OpenAIChatModelCompat
    openai_compatible_url = self.base_url.rstrip("/") + "/v1"
    return OpenAIChatModelCompat(
        model_name=model_id,
        stream=True,
        api_key=self.api_key,
        stream_tool_parsing=False,
        client_kwargs={"base_url": openai_compatible_url},
        generate_kwargs=self.generate_kwargs,
    )
```

### Model Download and Management

`LocalModelManager` in `manager.py` handles downloading from HuggingFace or ModelScope:

```python
# manager.py lines 94-120
class LocalModelManager:
    @staticmethod
    def download_model_sync(
        repo_id: str,
        filename: Optional[str] = None,
        backend: BackendType = BackendType.LLAMACPP,
        source: DownloadSource = DownloadSource.HUGGINGFACE,
    ) -> LocalModelInfo:
        # Auto-selects first GGUF file, preferring Q4_K_M quantization
        # Full repo download for MLX (needs all safetensors + config)
```

Auto-selection logic (lines 295-329):
- llama.cpp: Picks `.gguf` files, prefers `Q4_K_M` quantization
- MLX: Picks `.safetensors` files (triggers full repo snapshot download)

### Notable Technical Debt/Concerns

1. **Two separate Ollama systems**: `ollama_provider.py` (for chat) and `ollama_manager.py` (for model lifecycle) are separate. Model adds via `OllamaProvider.add_model()` actually trigger pulls, which is correct but the separation is confusing.

2. **MLX structured output fallback**: Since MLX lacks native JSON mode, the prompt gets a JSON schema appended. This is less reliable than true structured output.

3. **Manifest corruption handling**: `_load_manifest()` catches `JSONDecodeError` and `ValueError` but only logs a warning and returns fresh manifest - potential silent data loss.

---

## Feature 2: Desktop Application

### Implementation Overview

CoPaw's desktop application uses **pywebview** (a thin webview wrapper similar to Electron's approach but with native webview rendering) to embed the React console in a native window. The architecture spawns the FastAPI backend as a subprocess and displays the web UI in the native window.

### Key Files

```
src/copaw/cli/desktop_cmd.py    # Main desktop command implementation
scripts/pack/
├── build_common.py             # Shared conda-pack logic (macOS + Windows)
├── build_macos.sh              # macOS .app bundle builder
├── build_win.ps1               # Windows .exe NSIS installer builder
└── copaw_desktop.nsi           # NSIS installer template
```

### pywebview Integration

The `desktop_cmd()` in `desktop_cmd.py` implements the desktop workflow:

```python
# desktop_cmd.py lines 82-188
@click.command("desktop")
@click.option("--host", default="127.0.0.1")
@click.option("--log-level", default="info", type=click.Choice([...]))
def desktop_cmd(host: str, log_level: str) -> None:
    """Run CoPaw app on an auto-selected free port in a webview window."""
    port = _find_free_port(host)  # Bind to port 0 to get OS-assigned free port
    url = f"http://{host}:{port}"

    # Start FastAPI backend as subprocess
    proc = subprocess.Popen([
        sys.executable, "-m", "copaw", "app",
        "--host", host, "--port", str(port), "--log-level", log_level,
    ], stdin=subprocess.DEVNULL, ...)

    # Wait for HTTP server to be ready
    if _wait_for_http(host, port):
        # Create native window with JS API bridge
        api = WebViewAPI()
        webview.create_window("CoPaw Desktop", url, width=1280, height=800,
                               text_select=True, js_api=api)
        webview.start(private_mode=False)  # Blocks until window closed
```

### WebViewAPI Bridge

The `WebViewAPI` class exposes methods to the JavaScript side for handling external links:

```python
# desktop_cmd.py lines 29-36
class WebViewAPI:
    """API exposed to the webview for handling external links."""
    def open_external_link(self, url: str) -> None:
        """Open URL in system's default browser."""
        if not url.startswith(("http://", "https://")):
            return
        webbrowser.open(url)
```

This allows the console to open external links (like documentation) in the system browser rather than inside the webview.

### Subprocess Management

Notable attention to **subprocess cleanup and error handling**:

```python
# desktop_cmd.py lines 207-241
if proc and proc.poll() is None:  # process still running
    logger.info("Terminating backend server...")
    manually_terminated = True
    try:
        proc.terminate()
        try:
            proc.wait(timeout=5.0)
            logger.info("Backend server terminated cleanly.")
        except subprocess.TimeoutExpired:
            logger.warning("Backend did not exit in 5s, force killing...")
            proc.kill()
            proc.wait()
    except (ProcessLookupError, OSError) as e:
        # Process already exited, which is fine
        logger.debug(f"kill() raised {e.__class__.__name__} (process already exited)")
elif proc:
    logger.info(f"Backend already exited with code {proc.returncode}")
```

The code handles all race conditions between `poll()` and `terminate()`/`kill()`.

### Windows-specific: Stream Reader Threads

On Windows, subprocess output can block if not continuously drained. The code uses background threads:

```python
# desktop_cmd.py lines 61-79
def _stream_reader(in_stream, out_stream) -> None:
    """Read from in_stream line by line and write to out_stream.
    Used on Windows to prevent subprocess buffer blocking."""
    try:
        for line in iter(in_stream.readline, ""):
            if not line:
                break
            out_stream.write(line)
            out_stream.flush()
    except Exception:
        pass
```

### macOS Packaging (build_macos.sh)

The macOS build creates a `.app` bundle using conda-pack:

1. **Wheel build**: Builds `copaw-*.whl` including console frontend
2. **Conda-pack**: Creates portable Python environment
3. **Bundle structure**:
```
CoPaw.app/
└── Contents/
    ├── MacOS/CoPaw          # Launcher shell script
    ├── Resources/
    │   ├── env/             # Unpacked conda environment
    │   └── icon.icns        # App icon
    └── Info.plist           # macOS app metadata
```

**Launcher script** (lines 60-130 of `build_macos.sh`):
- Sets `PYTHONHOME` to bundled env
- Sets `COPAW_DESKTOP_APP=1` environment variable
- Preserves system `PATH` for commands like `imsg`, `brew`
- Sets SSL certificates from certifi
- Auto-runs `copaw init --defaults --accept-security` if no config exists
- Logs to `~/.copaw/desktop.log` when not a TTY

### Windows Packaging (build_win.ps1)

Windows uses NSIS installer with conda-pack:

1. **Zip archive** created by `build_common.py`
2. **Unpacked** to `win-unpacked/`
3. **conda-unpack.exe** rewrites path prefixes for portability
4. **conda-unpack bug workaround**: On Windows, conda-pack corrupts backslash escaping in Python strings (Issue #154). The script reinstalls affected packages (`huggingface_hub`) from cached wheels.
5. **Bytecode pre-compilation**: All `.py` files compiled to `.pyc` for faster startup
6. **NSIS installer**: Creates `CoPaw-Setup-VERSION.exe`

**Launcher setup**:
- `CoPaw Desktop.bat` - Main launcher (hidden via VBS)
- `CoPaw Desktop (Debug).bat` - Debug mode with visible console
- `CoPaw Desktop.vbs` - VBScript to hide console window

### SSL Certificate Handling

Both platforms query certifi for the SSL certificate location and set environment variables:

```bash
# macOS launcher
CERT_FILE=$("$ENV_DIR/bin/python" -c "import certifi; print(certifi.where())" 2>/dev/null)
if [ -n "$CERT_FILE" ] && [ -f "$CERT_FILE" ]; then
  export SSL_CERT_FILE="$CERT_FILE"
  export REQUESTS_CA_BUNDLE="$CERT_FILE"
  export CURL_CA_BUNDLE="$CERT_FILE"
fi
```

```powershell
# Windows launcher
"%~dp0python.exe" -u -c "import certifi; print(certifi.where())" > "%CERT_TMP%" 2>nul
set /p CERT_FILE=<"%CERT_TMP%"
if defined CERT_FILE (
  if exist "%CERT_FILE%" (
    set "SSL_CERT_FILE=%CERT_FILE%"
    set "REQUESTS_CA_BUNDLE=%CERT_FILE%"
    set "CURL_CA_BUNDLE=%CERT_FILE%"
  )
)
```

### Notable Technical Debt/Concerns

1. **NSIS installer complexity**: The PowerShell build script is ~317 lines with extensive debugging output and multiple fallback paths. The Windows build is significantly more complex than macOS.

2. **conda-unpack bug workaround**: The Windows build has explicit code to work around a known conda-pack bug. This adds maintenance burden and makes the build fragile.

3. **No auto-update mechanism**: The desktop app doesn't have an integrated update system - users must download new installers manually.

4. **pywebview limitations**: The app uses `private_mode=False` which may have security implications for shared computers.

---

## Feature 3: CLI (Command-Line Interface)

### Implementation Overview

CoPaw's CLI uses **Click** with a sophisticated **lazy loading pattern** that defers import of heavy subcommands until they're actually invoked. The main entry point registers 19 subcommands with lazy loading.

### Key Files

```
src/copaw/cli/
├── main.py          # CLI entry point with LazyGroup
├── init_cmd.py      # Interactive initialization
├── agents_cmd.py    # Agent management
├── channels_cmd.py  # Channel configuration (35KB - largest)
├── chats_cmd.py     # Chat management
├── cron_cmd.py      # Cron/scheduled task management
├── desktop_cmd.py   # Desktop app launcher
├── env_cmd.py       # Environment variable management
├── providers_cmd.py # LLM provider + model management (889 lines)
├── skills_cmd.py    # Skills management
├── update_cmd.py    # Update management
├── shutdown_cmd.py  # Shutdown command
├── uninstall_cmd.py
├── auth_cmd.py
├── clean_cmd.py
├── daemon_cmd.py
├── utils.py         # Shared CLI utilities
└── process_utils.py
```

### Lazy Loading Pattern

The `LazyGroup` class enables lazy loading of subcommands:

```python
# main.py lines 55-89
class LazyGroup(click.Group):
    """Click group that supports lazy loading of subcommands."""
    def __init__(self, *args, lazy_subcommands=None, **kwargs):
        super().__init__(*args, **kwargs)
        self.lazy_subcommands = lazy_subcommands or {}

    def get_command(self, ctx, cmd_name):
        """Get command, loading lazily if needed."""
        # Try eager commands first
        cmd = super().get_command(ctx, cmd_name)
        if cmd is not None:
            return cmd

        # Try lazy commands - import module only when needed
        if cmd_name in self.lazy_subcommands:
            module_path, attr_name, label = self.lazy_subcommands[cmd_name]
            _t = time.perf_counter()
            try:
                module = __import__(module_path, fromlist=[attr_name])
                cmd = getattr(module, attr_name)
                _record(label, time.perf_counter() - _t)
                self.add_command(cmd, cmd_name)  # Cache for next time
                return cmd
            except Exception as e:
                logger.error(f"Failed to load command '{cmd_name}': {e}")
                return None
        return None
```

Registration (lines 92-135):
```python
@click.group(
    cls=LazyGroup,
    context_settings={"help_option_names": ["-h", "--help"]},
    lazy_subcommands={
        "app": ("copaw.cli.app_cmd", "app_cmd", ".app_cmd"),
        "channels": ("copaw.cli.channels_cmd", "channels_group", ".channels_cmd"),
        # ... 17 more subcommands
    },
)
```

### Init Command (init_cmd.py)

The `init` command is a comprehensive interactive setup wizard covering:

1. **Security warning** - Rich-formatted panel with security considerations
2. **Telemetry opt-in** - Anonymous usage data collection
3. **Workspace initialization** - Creates default agent workspace
4. **Heartbeat configuration** - Interval, target agent, active hours
5. **Show tool details** - Whether to display tool call details in messages
6. **Language selection** - zh/en/ru for MD files
7. **Audio mode** - auto vs native transcription
8. **Channel configuration** - Interactive channel setup
9. **LLM provider setup** - Provider + model selection
10. **Skills sync** - Enable all, none, or custom selection
11. **Environment variables** - Interactive env var configuration
12. **MD files** - Copy language-specific markdown files
13. **HEARTBEAT.md** - Creates default heartbeat query file

Notable: Supports `--defaults` and `--accept-security` flags for non-interactive/scripted initialization:

```python
# init_cmd.py lines 119-130
@click.command("init")
@click.option("--force", is_flag=True, help="Overwrite existing config.json...")
@click.option("--defaults", "use_defaults", is_flag=True,
              help="Use defaults only, no interactive prompts (for scripts).")
@click.option("--accept-security", is_flag=True,
              help="Skip security confirmation (use with --defaults for scripts/Docker).")
```

### Providers Command (providers_cmd.py)

The most complex CLI command at ~889 lines with extensive provider management:

**Subcommands**:
- `models list` - Show all providers and configuration
- `models config` - Interactive provider + model configuration
- `models config-key` - Configure API key for a provider
- `models set-llm` - Set active LLM model
- `models add-provider` - Add custom OpenAI-compatible provider
- `models remove-provider` - Remove custom provider
- `models add-model` - Add model to provider
- `models remove-model` - Remove model from provider
- `models download` - Download GGUF model from HuggingFace/ModelScope
- `models local` - List downloaded local models
- `models remove-local` - Delete downloaded local model
- `models ollama-pull` - Pull Ollama model
- `models ollama-list` - List Ollama models
- `models ollama-remove` - Delete Ollama model

**Interactive provider selection with status indicators**:
```python
# providers_cmd.py lines 66-92
def _select_provider_interactive(prompt_text, *, default_pid) -> str:
    all_providers = _all_provider_objects(manager)
    labels, ids = [], []
    for provider in all_providers:
        mark = "✓" if _is_configured(provider) else "✗"
        labels.append(f"{provider.name} ({provider.id}) [{mark}]")
        ids.append(provider.id)
    chosen_label = prompt_choice(prompt_text, options=labels, ...)
    return ids[labels.index(chosen_label)]
```

**Local vs API provider handling**:
```python
# providers_cmd.py lines 28-37
def _is_configured(provider: Provider) -> bool:
    if provider.is_local or provider.id == "ollama":
        return True  # Local providers always "configured"
    if not provider.base_url:
        return False
    if provider.require_api_key and not provider.api_key:
        return False
    return True
```

### Shared CLI Utilities (utils.py)

The `prompt_choice` and `prompt_confirm` helpers wrap Click with Rich for pretty prompts:

```python
# utils.py
def prompt_choice(prompt_text: str, options: list[str], default: str | None = None) -> str:
    """Prompt user to select from a list of options using click.Choice."""
    ...

def prompt_confirm(prompt_text: str, default: bool = False) -> bool:
    """Ask for yes/no confirmation."""
    ...
```

### Timing Instrumentation

The main CLI module tracks import times for performance monitoring:

```python
# main.py lines 23-46
_init_timings: list[tuple[str, float]] = []
_t0_main = time.perf_counter()
_init_timings.append(("main.py loaded", 0.0))

def _record(label: str, elapsed: float) -> None:
    _init_timings.append((label, elapsed))
    logger.debug("%.3fs %s", elapsed, label)

# Timed imports
_t = time.perf_counter()
from ..config.utils import read_last_api
_record("..config.utils", time.perf_counter() - _t)
```

This allows debugging slow CLI startup by running with `--log-level debug`.

### UTF-8 Handling on Windows

```python
# main.py lines 10-17
if sys.platform == "win32":
    try:
        sys.stdout.reconfigure(encoding="utf-8")
        sys.stderr.reconfigure(encoding="utf-8")
    except (AttributeError, OSError):
        pass
```

### Notable Technical Debt/Concerns

1. **Massive channels_cmd.py** (~35KB): The channels command is very large, likely containing all channel-specific configuration. Should be split into per-channel modules.

2. **Inconsistent provider vs model vs local model**: The `models` group conflates three different concepts:
   - API providers (OpenAI, Anthropic, etc.)
   - Local llama.cpp/MLX model files
   - Ollama daemon models
   These have different management paradigms and the CLI doesn't fully hide this complexity.

3. **No validation on provider add**: `add-provider` accepts any values without validation of the base URL format or API key prefix.

4. **Deep nesting**: The CLI has `models ollama-remove` which is 3 levels deep. Some commands like `copaw models remove-local model_id` could be confused since `model_id` format differs between local and Ollama.

---

## Summary of Architectural Patterns

| Feature | Pattern | Strength | Concern |
|---------|---------|----------|---------|
| Local Models | Singleton backend + abstract factory | Fast model reuse, clean separation | Manifest corruption silently ignored |
| Local Models | Tag parsing for raw outputs | Handles local model quirks | Streaming incomplete tag handling is complex |
| Desktop | pywebview + subprocess | Zero-config, cross-platform | No auto-update, private_mode=False |
| Desktop | conda-pack for env bundling | Full dependency isolation | conda-unpack bug workaround is fragile |
| CLI | Lazy loading via LazyGroup | Fast startup for unused commands | Error messages during lazy load can be cryptic |
| CLI | Rich-based interactive prompts | Pretty UX | Different from standard Click look |
