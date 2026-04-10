# Feature Deep Dive - Batch 5: Docker Deployment & Voice/Audio Support

## 1. Docker Deployment

### Implementation Overview

CoPaw provides containerized deployment via Docker with a multi-stage Dockerfile and docker-compose configuration. The deployment is designed to run a full desktop-like environment inside the container.

#### Key Files

| File | Purpose |
|------|---------|
| `deploy/Dockerfile` | Multi-stage build: console frontend build + Python runtime |
| `docker-compose.yml` (root) | User-facing compose for running from Docker Hub image |
| `deploy/entrypoint.sh` | Process supervisor startup script |
| `deploy/config/supervisord.conf.template` | Supervisor configuration template |

#### Architecture

**Multi-Stage Build Pattern:**

```dockerfile
# Stage 1: Build console frontend
FROM agentscope-registry.ap-southeast-1.cr.aliyuncs.com/agentscope/node:slim AS console-builder
WORKDIR /app
COPY console /app/console
RUN cd /app/console && npm ci --include=dev && npm run build

# Stage 2: Runtime image
FROM agentscope-registry.ap-southeast-1.cr.aliyuncs.com/agentscope/node:slim
# ... runtime setup ...
COPY --from=console-builder /app/console/dist/ ./src/copaw/console/
```

**Process Supervisor Architecture:**

The container runs multiple processes managed by `supervisord`:

```
[supervisord]
  ├── [program:dbus]      - D-Bus system daemon
  ├── [program:xvfb]       - X Virtual Framebuffer (headless X11)
  ├── [program:xfce4]      - XFCE4 desktop environment
  └── [program:app]       - CoPaw FastAPI application
```

The `entrypoint.sh` substitutes `COPAW_PORT` in the supervisor template at runtime:

```bash
set -e
export COPAW_PORT="${COPAW_PORT:-8088}"
envsubst '${COPAW_PORT}' \
  < /etc/supervisor/conf.d/supervisord.conf.template \
  > /etc/supervisor/conf.d/supervisord.conf
exec /usr/bin/supervisord -c /etc/supervisor/conf.d/supervisord.conf
```

#### Volume Mounts

```yaml
volumes:
  copaw-data:/app/working       # Working directory for app data
  copaw-secrets:/app/working.secret  # Secrets directory
```

#### Environment Variables for Channel Management

```dockerfile
ARG COPAW_DISABLED_CHANNELS="imessage"
ARG COPAW_ENABLED_CHANNELS=""
```

Users can override at runtime: `-e COPAW_DISABLED_CHANNELS="imessage,voice"`

#### Browser/Playwright Configuration

```dockerfile
RUN sed -i 's/^CHROMIUM_FLAGS=""/CHROMIUM_FLAGS="--no-sandbox"/' /usr/bin/chromium

ENV PLAYWRIGHT_CHROMIUM_EXECUTABLE_PATH=/usr/bin/chromium
ENV PLAYWRIGHT_SKIP_BROWSER_DOWNLOAD=1
ENV COPAW_RUNNING_IN_CONTAINER=1
```

The `--no-sandbox` flag is patched directly into the chromium launch script to work in container environment.

### Notable Code Patterns

1. **No-commit dist pattern**: The console `dist/` is not committed to the repo. Stage 1 builds it from source during Docker build.

2. **Init at build time**: `RUN copaw init --defaults --accept-security` initializes the working directory during image build, not at container startup.

3. **Lazy initialization**: The venv is created at build time but the app initialization happens during the build stage.

### Technical Debt / Concerns

1. **Aliyun registry dependency**: The base images are hosted on Aliyun registry (`agentscope-registry.ap-southeast-1.cr.aliyuncs.com`). This could be problematic for users in non-China regions due to latency and access restrictions.

2. **Supervisor process chain**: The dependency chain (`xvfb` -> `dbus` -> `xfce4` -> `app`) with priority values (10, 20, 30) is fragile. The xfce4 startup waits up to 20 seconds for X11 socket:

   ```bash
   command=sh -c 'export DISPLAY=:1; for i in $(seq 1 200); do [ -S /tmp/.X11-unix/X1 ] && break; sleep 0.1; done; exec dbus-run-session startxfce4'
   ```

3. **Chromium patching via sed**: Modifying system binaries via `sed -i` is fragile and may break on package updates.

4. **Single-stage runtime**: Despite building the console in a separate stage, the final image is large due to Chromium, XFCE4, and other desktop environment dependencies (~2GB+ estimated).

5. **Image not on Docker Hub**: The compose references `agentscope/copaw:latest` on Docker Hub but the actual Dockerfile builds from Aliyun registry. Users building locally would need to push to their own registry.

---

## 2. Voice/Audio Support

### Implementation Overview

CoPaw's voice channel provides phone-call-based interaction via Twilio's ConversationRelay API combined with Cloudflare Quick Tunnels for webhook delivery.

#### Key Files

| File | Purpose |
|------|---------|
| `src/copaw/app/channels/voice/channel.py` | Main VoiceChannel class |
| `src/copaw/app/channels/voice/conversation_relay.py` | WebSocket handler for Twilio |
| `src/copaw/app/channels/voice/session.py` | Call session management |
| `src/copaw/app/channels/voice/twilio_manager.py` | Twilio API wrapper |
| `src/copaw/app/channels/voice/twiml.py` | TwiML generation helpers |
| `src/copaw/app/routers/voice.py` | FastAPI endpoints |
| `src/copaw/tunnel/cloudflare.py` | Cloudflare tunnel driver |

#### Architecture

**Call Flow:**

```
Phone -> Twilio -> [Cloudflare Quick Tunnel] -> CoPaw /voice/incoming (webhook)
                                              -> CoPaw /voice/ws (WebSocket)

CoPaw [agent response] -> WebSocket -> Twilio -> [TTS] -> Phone
```

**Startup Sequence (channel.py `start()`):**

```python
async def start(self) -> None:
    # 1. Start Cloudflare tunnel
    self.tunnel_mgr = CloudflareTunnelDriver()
    tunnel_info = await self.tunnel_mgr.start(local_port)

    # 2. Configure Twilio webhook URL
    base_url = tunnel_info.public_url
    webhook_url = f"{base_url}/voice/incoming"
    status_cb_url = f"{base_url}/voice/status-callback"
    await self.twilio_mgr.configure_voice_webhook(
        phone_number_sid,
        webhook_url,
        status_callback_url=status_cb_url,
    )
```

**WebSocket Message Protocol:**

Twilio -> CoPaw messages:
```python
{"type":"setup", "callSid":"CA...", "from":"+1...", "to":"+2..."}
{"type":"prompt", "voicePrompt":"transcribed text"}
{"type":"interrupt", "utteranceUntilInterrupt":"user speech before interrupt"}
{"type":"dtmf", "digit":"5"}
```

CoPaw -> Twilio messages:
```python
{"type":"text", "token": "response chunk", "last": false}
{"type":"text", "token": "", "last": true}  # Signal end of response
{"type": "end"}  # End call
```

**Single-Use Token Security:**

```python
def create_ws_token(self) -> str:
    # Evict oldest if dict grows too large (prevents memory leak)
    while len(self._pending_ws_tokens) >= self._MAX_PENDING_TOKENS:
        self._pending_ws_tokens.popitem(last=False)
    token = secrets.token_urlsafe(32)
    self._pending_ws_tokens[token] = None
    return token

def validate_ws_token(self, token: str) -> bool:
    # Pop returns ... if key not found (Ellipsis singleton)
    return self._pending_ws_tokens.pop(token, ...) is not ...
```

The token is created at `/voice/incoming` and validated at `/voice/ws`. This prevents direct WebSocket connection attempts without going through Twilio.

**Twilio Signature Validation:**

```python
async def _validate_twilio_signature(request: Request) -> None:
    # Behind a tunnel/reverse-proxy, reconstruct the public URL
    # using forwarded headers so signature validation succeeds
    proto = request.headers.get("x-forwarded-proto", request.url.scheme)
    host = request.headers.get("x-forwarded-host", request.url.netloc)
    url = f"{proto}://{host}{request.url.path}"
    # ...
    validator = RequestValidator(auth_token)
    if not validator.validate(url, params, signature):
        raise HTTPException(status_code=403, detail="Invalid Twilio signature")
```

**TwiML Generation (twiml.py):**

```python
def build_conversation_relay_twiml(
    ws_url: str,
    welcome_greeting: str = "Hi! This is CoPaw. How can I help you?",
    tts_provider: str = "google",
    tts_voice: str = "en-US-Journey-D",
    stt_provider: str = "deepgram",
    language: str = "en-US",
    interruptible: bool = True,
) -> str:
    response_el = ET.Element("Response")
    connect_el = ET.SubElement(response_el, "Connect")
    ET.SubElement(
        connect_el,
        "ConversationRelay",
        url=ws_url,
        welcomeGreeting=welcome_greeting,
        ttsProvider=tts_provider,
        voice=tts_voice,
        transcriptionProvider=stt_provider,
        language=language,
        interruptible=str(interruptible).lower(),
    )
```

**Cloudflare Tunnel Driver (cloudflare.py):**

The tunnel parses `cloudflared` stderr output to extract the public URL:

```python
_URL_RE = re.compile(r"https://[a-zA-Z0-9\-]+\.trycloudflare\.com")

async def _wait_for_url(self, timeout: float = 30) -> str | None:
    # Read stderr until URL appears
    while loop.time() < deadline:
        line = await asyncio.wait_for(self._process.stderr.readline(), ...)
        match = _URL_RE.search(line.decode("utf-8", errors="replace"))
        if match:
            return match.group(0)
```

The tunnel URL is converted to WSS by replacing `https://` with `wss://`.

### Session Management

```python
@dataclass
class CallSession:
    call_sid: str
    from_number: str
    to_number: str
    started_at: datetime
    handler: ConversationRelayHandler
    status: str = "active"  # active | ended | failed

class CallSessionManager:
    def __init__(self) -> None:
        self._sessions: dict[str, CallSession] = {}

    def active_sessions(self) -> list[CallSession]:
        return [s for s in self._sessions.values() if s.status == "active"]
```

### Response Streaming

The agent response is streamed token-by-token to Twilio for immediate TTS playback:

```python
async def _process_and_stream(self, request: Any) -> None:
    async for event in self._process(request):
        if obj == "message" and status == RunStatus.Completed:
            text = self._extract_text_from_event(event)
            if text:
                # Send token immediately
                await self._send_token(text, last=False)
                # Signal end of this message for TTS
                await self._send_token("", last=True)
```

This allows Twilio to start TTS playback before the entire agent response is complete.

### Error Handling

```python
_ERROR_MSG = "I'm having trouble right now. Please try again."

async def _process_and_stream(self, request: Any) -> None:
    except Exception:
        logger.exception("Error processing voice request: call_sid=%s", self.call_sid)
        if not self._closed:
            await self._send_token(_ERROR_MSG, last=False)
            await self._send_token("", last=True)
```

### Configuration Options

From `build_conversation_relay_twiml()` and `from_config()`:
- `welcome_greeting`: Initial greeting message
- `tts_provider`: "google" (default) or other Twilio-supported providers
- `tts_voice`: Voice name (e.g., "en-US-Journey-D")
- `stt_provider`: "deepgram" (default) or other Twilio-supported providers
- `language`: e.g., "en-US"
- `interruptible`: Whether user can interrupt the AI (default: True)

### Technical Debt / Concerns

1. **Quick Tunnel limitations**: Cloudflare Quick Tunnels (`*.trycloudflare.com`) are designed for temporary/development use. They:
   - Change the URL each time the tunnel restarts
   - Have rate limits
   - Are not guaranteed to be always available
   - The comment in `_monitor()` acknowledges: "not restarting automatically because a new public URL would be issued"

2. **Twilio credentials in config**: The `twilio_account_sid` and `twilio_auth_token` are stored in the config file. For production, these should be environment variables or secrets management.

3. **No reconnection handling**: If the tunnel dies, Twilio webhook delivery fails and calls cannot be received. There's no automatic reconnection or alerting.

4. **Interrupt handling is incomplete**: The comment notes:
   ```python
   # We log the interruption but don't truncate context.
   # Future: truncate assistant message to what was actually spoken.
   ```

5. **DTMF handled but not acted upon**: DTMF digits are logged but no skill or action is triggered by them.

6. **No concurrent call limits**: The `CallSessionManager` doesn't enforce any limit on concurrent calls. A busy voice channel could spawn unlimited sessions.

7. **Binary download dependency**: `cloudflared` binary is downloaded on first use via `BinaryManager`. If the download fails, voice channel won't start.

### Clever Solutions

1. **Forwarded header reconstruction**: The Twilio signature validation correctly handles being behind a tunnel by reconstructing the public URL from `x-forwarded-*` headers.

2. **Ellipsis sentinel for token validation**: Using `self._pending_ws_tokens.pop(token, ...) is not ...` is a concise way to check for key existence while popping in one operation.

3. **Token eviction to prevent memory leak**: The `_MAX_PENDING_TOKENS = 100` limit with oldest eviction prevents unbounded memory growth if TwiML is generated but WebSocket never connects.

4. **Streaming TTS**: Sending tokens incrementally with `last=true` markers allows Twilio's TTS to start speaking before the full response is ready, reducing perceived latency.
