# PicoClaw Features Batch 7 - Voice Pipeline & Cross-Platform Binary

## Feature 18: Voice Pipeline

### Overview

The voice pipeline enables audio-to-text transcription of voice messages before they reach the AI agent. The agent sees transcribed text instead of audio references, allowing it to respond to voice input naturally.

### Architecture

The voice pipeline uses an interface-based design with provider-specific implementations:

```
pkg/voice/
├── transcriber.go              # Interface + provider detection
├── audio_model_transcriber.go   # Multimodal LLM transcription
├── elevenlabs_transcriber.go   # ElevenLabs Scribe API
├── groq_transcriber.go         # Groq Whisper API
└── *_test.go                   # Tests for each implementation
```

### Core Interface

**File:** `pkg/voice/transcriber.go`

```go
type Transcriber interface {
    Name() string
    Transcribe(ctx context.Context, audioFilePath string) (*TranscriptionResponse, error)
}

type TranscriptionResponse struct {
    Text     string  `json:"text"`
    Language string  `json:"language,omitempty"`
    Duration float64 `json:"duration,omitempty"`
}
```

### Provider Detection Logic

The `DetectTranscriber()` function implements a priority-based selection:

1. **Voice model name** (highest priority) - If `cfg.Voice.ModelName` is set and the model supports audio transcription, use `AudioModelTranscriber`
2. **ElevenLabs API key** - If `cfg.Voice.ElevenLabsAPIKey` is set, use `ElevenLabsTranscriber`
3. **Groq model fallback** - If any model in `cfg.ModelList` starts with `groq/` and has an API key, use `GroqTranscriber`

```go
// From pkg/voice/transcriber.go lines 46-68
func DetectTranscriber(cfg *config.Config) Transcriber {
    if modelName := strings.TrimSpace(cfg.Voice.ModelName); modelName != "" {
        modelCfg, err := cfg.GetModelConfig(modelName)
        if err != nil {
            return nil
        }
        if supportsAudioTranscription(modelCfg.Model) {
            return NewAudioModelTranscriber(modelCfg)
        }
    }

    if key := strings.TrimSpace(cfg.Voice.ElevenLabsAPIKey); key != "" {
        return NewElevenLabsTranscriber(key)
    }

    for _, mc := range cfg.ModelList {
        if strings.HasPrefix(mc.Model, "groq/") && mc.APIKey() != "" {
            return NewGroqTranscriber(mc.APIKey())
        }
    }
    return nil
}
```

### Supported Protocols for Audio Model Transcription

The `supportsAudioTranscription()` function (lines 22-42) whitelists these protocols:

- openai, azure, azure-openai, litellm, openrouter, groq, zhipu, gemini, nvidia
- ollama, moonshot, shengsuanyun, deepseek, cerebras
- vivgrid, volcengine, vllm, qwen, qwen-intl, qwen-international
- qwen-us, dashscope-us, mistral, avian, minimax, longcat, modelscope, novita
- coding-plan, alibaba-coding, qwen-coding

**Technical Debt Note:** The TODO comment at line 36-37 acknowledges this is overly broad:
```go
// TODO: Further restrict this by modelID, since not every model under these
// protocols supports audio transcription.
```

### Transcriber Implementations

#### 1. AudioModelTranscriber (Multimodal LLM)

**File:** `pkg/voice/audio_model_transcriber.go`

Uses the provider's standard Chat API with audio media:

```go
resp, err := t.provider.Chat(ctx, []providers.Message{
    {
        Role:    "user",
        Content: t.prompt,
        Media: []string{
            fmt.Sprintf("data:audio/%s;base64,%s", format, base64.StdEncoding.EncodeToString(audioBytes)),
        },
    },
}, nil, t.modelID, map[string]any{
    "temperature": 0,
})
```

Key characteristics:
- Reads audio file, detects format via `utils.AudioFormat()`
- Base64 encodes audio and sends as data URI
- Uses temperature 0 for deterministic transcription
- Returns only `Text` field (Language/Duration from provider ignored)

#### 2. ElevenLabsTranscriber

**File:** `pkg/voice/elevenlabs_transcriber.go`

Uses the ElevenLabs Scribe API v1:

```go
func (t *ElevenLabsTranscriber) Transcribe(ctx context.Context, audioFilePath string) (*TranscriptionResponse, error) {
    // Opens audio file, creates multipart form request
    // POST to https://api.elevenlabs.io/v1/speech-to-text
    // Uses "scribe_v1" model
    // 120 second timeout
}
```

Key characteristics:
- Multipart form upload with `file` and `model_id` fields
- Hardcoded to `scribe_v1` model
- 120 second HTTP timeout

#### 3. GroqTranscriber

**File:** `pkg/voice/groq_transcriber.go`

Uses Groq's OpenAI-compatible transcription endpoint:

```go
if err = writer.WriteField("model", "whisper-large-v3"); err != nil { ... }
if err = writer.WriteField("response_format", "json"); err != nil { ... }
```

Key characteristics:
- Hardcoded to `whisper-large-v3` model
- 60 second HTTP timeout
- OpenAI-compatible API format

### Configuration

**File:** `pkg/config/config.go` lines 835-839

```go
type VoiceConfig struct {
    ModelName         string `json:"model_name,omitempty"         env:"PICOCLAW_VOICE_MODEL_NAME"`
    EchoTranscription bool   `json:"echo_transcription"           env:"PICOCLAW_VOICE_ECHO_TRANSCRIPTION"`
    ElevenLabsAPIKey  string `json:"elevenlabs_api_key,omitempty" env:"PICOCLAW_VOICE_ELEVENLABS_API_KEY"`
}
```

### Agent Integration

**File:** `pkg/agent/loop.go` lines 1056-1080

The `transcribeAudioInMessage()` method:

1. Checks if transcriber and media store exist
2. Iterates through message media refs
3. Resolves each ref to a file path
4. Skips non-audio files (uses `utils.IsAudioFile()`)
5. Calls transcriber and appends result to transcriptions array
6. Replaces audio annotations in message content with transcribed text

```go
func (al *AgentLoop) transcribeAudioInMessage(ctx context.Context, msg bus.InboundMessage) (bus.InboundMessage, bool) {
    if al.transcriber == nil || al.mediaStore == nil || len(msg.Media) == 0 {
        return msg, false
    }

    var transcriptions []string
    for _, ref := range msg.Media {
        path, meta, err := al.mediaStore.ResolveWithMeta(ref)
        if err != nil {
            logger.WarnCF("voice", "Failed to resolve media ref", map[string]any{"ref": ref, "error": err})
            continue
        }
        if !utils.IsAudioFile(meta.Filename, meta.ContentType) {
            continue
        }
        result, err := al.transcriber.Transcribe(ctx, path)
        // ...
    }
    // Replaces audio refs in msg.Content with transcribed text
}
```

**Called from main loop at line 580:**
```go
// Transcribe audio if needed before steering, so the agent sees text.
msg, _ = al.transcribeAudioInMessage(ctx, msg)
```

### Error Handling

- Media resolve failures: Logged as warnings, skipped gracefully
- Transcription failures: Logged, empty string appended to transcriptions
- No transcriber: Audio messages pass through unchanged (agent receives audio refs)

### Test Coverage

Comprehensive tests in `pkg/voice/transcriber_test.go` cover:
- No config returns nil
- Voice model name with OpenAI-compatible protocol selects audio-model
- Voice model name with non-compatible protocol (e.g., anthropic) returns nil
- ElevenLabs key selection
- ElevenLabs takes priority over groq fallback
- Voice model name takes priority over ElevenLabs
- Groq model list fallback
- Missing API keys are handled (returns nil)

---

## Feature 20: Cross-Platform Binary

### Overview

PicoClaw is distributed as a single static binary across x86_64, ARM64, ARMv6/7, RISC-V64, MIPS, and LoongArch architectures. The build system uses GoReleaser for releases and a comprehensive Makefile for development builds.

### Build Configuration

#### GoReleaser (`.goreleaser.yaml`)

Defines 3 binaries for release:
1. `picoclaw` - Main CLI agent (`cmd/picoclaw`)
2. `picoclaw-launcher` - Web console (`web/backend`)
3. `picoclaw-launcher-tui` - Terminal UI (`cmd/picoclaw-launcher-tui`)

**Build Environment:**
```yaml
env:
  - CGO_ENABLED=0  # Static binaries, no cgo dependencies
tags:
  - goolm          # JSON stdlib optimization
  - stdjson
ldflags:
  - -s -w          # Strip symbols
  - -X github.com/sipeed/picoclaw/pkg/config.Version={{ .Version }}
  - -X github.com/sipeed/picoclaw/pkg/config.GitCommit={{ .ShortCommit }}
  - -X github.com/sipeed/picoclaw/pkg/config.BuildTime={{ .Date }}
  - -X github.com/sipeed/picoclaw/pkg/config.GoVersion={{ .Env.GOVERSION }}
```

#### Supported Platforms

**Operating Systems:**
- linux
- windows
- darwin
- freebsd
- netbsd

**Architectures:**
```yaml
goarch:
  - amd64      # x86_64
  - arm64      # ARM64 / AArch64
  - riscv64    # RISC-V 64-bit
  - loong64    # LoongArch 64-bit
  - arm        # ARMv6/7 (32-bit)
  - s390x      # IBM Z
  - mipsle     # MIPS Little-Endian (32-bit)
goarm:
  - "6"        # ARMv6 (Raspberry Pi 1/Zero)
  - "7"        # ARMv7 (Cortex-A7)
gomips:
  - softfloat  # MIPS soft float (no FPU)
```

**Excluded Combinations:**
```yaml
ignore:
  - goos: windows
    goarch: arm
  - goos: netbsd
    goarch: s390x
  - goos: netbsd
    goarch: mips64
  - goos: netbsd
    goarch: arm
```

### Makefile Build Targets

**File:** `Makefile`

| Target | Purpose |
|--------|---------|
| `build` | Build for current platform |
| `build-all` | Build for all platforms (linux-amd64, linux-arm, linux-arm64, linux-loong64, linux-riscv64, linux-mipsle, darwin-arm64, windows-amd64, netbsd-amd64, netbsd-arm64) |
| `build-launcher` | Build web console binary |
| `build-launcher-tui` | Build TUI launcher binary |
| `build-whatsapp-native` | Build with WhatsApp native (larger binary) |
| `build-linux-arm` | Build for Linux ARMv7 |
| `build-linux-arm64` | Build for Linux ARM64 |
| `build-linux-mipsle` | Build for Linux MIPS32 LE |
| `build-pi-zero` | Build for Raspberry Pi Zero 2 W (both 32/64-bit) |
| `docker-build` | Build minimal Alpine Docker image |
| `docker-build-full` | Build full-featured Docker with Node.js 24 |

### Platform-Specific Build Logic

#### Linux Detection (lines 77-93)
```makefile
ifeq ($(UNAME_S),Linux)
    PLATFORM=linux
    ifeq ($(UNAME_M),x86_64)
        ARCH=amd64
    else ifeq ($(UNAME_M),aarch64)
        ARCH=arm64
    else ifeq ($(UNAME_M),loongarch64)
        ARCH=loong64
    else ifeq ($(UNAME_M),riscv64)
        ARCH=riscv64
    else ifeq ($(UNAME_M),mipsel)
        ARCH=mipsle
    endif
```

#### macOS Special Handling (lines 94-103)
```makefile
else ifeq ($(UNAME_S),Darwin)
    PLATFORM=darwin
    WEB_GO=CGO_ENABLED=1 go  # Web launcher requires cgo on macOS
    ifeq ($(UNAME_M),x86_64)
        ARCH=amd64
    else ifeq ($(UNAME_M),arm64)
        ARCH=arm64
    endif
```

### Special Build Fixes

#### MIPS ELF Flags Patch (lines 28-48)

For kernels that only support NaN2008 IEEE 754 encoding:

```makefile
define PATCH_MIPS_FLAGS
    @if [ -f "$(1)" ]; then \
        printf '\004\024\000\160' | dd of=$(1) bs=1 seek=36 count=4 conv=notrunc 2>/dev/null || \
        { echo "Error: failed to patch MIPS e_flags for $(1)"; exit 1; }; \
    fi
endef
```

This patches byte offset 36 in the ELF header with:
- `0x70000000` - EF_MIPS_ARCH_32R2 (MIPS32 Release 2)
- `0x00001000` - EF_MIPS_ABI_O32 (O32 ABI)
- `0x00000400` - EF_MIPS_NAN2008 (IEEE 754-2008 NaN)
- `0x00000004` - EF_MIPS_CPIC (PIC calling sequence)

Used in:
```makefile
GOOS=linux GOARCH=mipsle GOMIPS=softfloat $(GO) build ... -o $(BUILD_DIR)/$(BINARY_NAME)-linux-mipsle
$(call PATCH_MIPS_FLAGS,$(BUILD_DIR)/$(BINARY_NAME)-linux-mipsle)
```

#### Loong64 PTY Patch (lines 50-55)

The `creack/pty` package lacks `ztypes_loong64.go`:

```makefile
define PTY_PATCH_LOONG64
    @pty_dir=$$(go env GOMODCACHE)/github.com/creack/pty@v1.1.9; \
    if [ -d "$$pty_dir" ] && [ ! -f "$$pty_dir/ztypes_loong64.go" ]; then \
        printf '//go:build linux && loong64\npackage pty\ntype (_C_int int32; _C_uint uint32)\n' > "$$pty_dir/ztypes_loong64.go"; \
    fi
endef
```

### Docker Builds

**File:** `docker/Dockerfile.goreleaser`

Two profiles in `docker/docker-compose.yml`:
1. `picoclaw-agent` - Minimal Alpine-based agent
2. `picoclaw-gateway` - Gateway mode with port exposed

Platforms built:
- linux/amd64
- linux/arm64
- linux/riscv64

### Release Artifacts

**Archive formats:**
- tar.gz for Linux/macOS
- zip for Windows

**Naming template:**
```
{{ .ProjectName }}_{{ title .Os }}_{{ if eq .Arch "amd64" }}x86_64{{ else if eq .Arch "386" }}i386{{ else }}{{ .Arch }}{{ end }}{{ if .Arm }}v{{ .Arm }}{{ end }}
```

**Package formats (nfpms):**
- rpm
- deb

Includes desktop file and icon for Linux desktop integration.

### macOS Signing & Notarization

**File:** `.goreleaser.yaml` lines 174-189

```yaml
notarize:
  macos:
    - enabled: '{{ isEnvSet "MACOS_SIGN_P12" }}'
      ids:
        - picoclaw
        - picoclaw-launcher
        - picoclaw-launcher-tui
      sign:
        certificate: "{{.Env.MACOS_SIGN_P12}}"
        password: "{{.Env.MACOS_SIGN_PASSWORD}}"
      notarize:
        issuer_id: "{{.Env.MACOS_NOTARY_ISSUER_ID}}"
        key_id: "{{.Env.MACOS_NOTARY_KEY_ID}}"
        key: "{{.Env.MACOS_NOTARY_KEY}}"
        wait: true
        timeout: 20m
```

Requires environment variables:
- `MACOS_SIGN_P12` - Signing certificate
- `MACOS_SIGN_PASSWORD` - Certificate password
- `MACOS_NOTARY_ISSUER_ID` - Notarization issuer
- `MACOS_NOTARY_KEY_ID` - Notarization key ID
- `MACOS_NOTARY_KEY` - Notarization private key

### Verified Hardware

**File:** `docs/hardware-compatibility.md`

**x86:** Intel/AMD any x86 CPU

**ARM:**
- ARMv6: BCM2835 (Raspberry Pi 1/Zero)
- ARMv7: Allwinner V3s (LicheePi Zero)
- ARM64: BCM2711/2712 (Raspberry Pi 4/5), Allwinner H618 (Orange Pi Zero 3), AX630C (NanoKVM-Pro, MaixCAM2)

**RISC-V:**
- SOPHGO SG2002 (NanoKVM, LicheeRV-Nano)
- Allwinner V861, V881
- SpacemiT K1, K3 (Milk-V Jupiter, BananaPi BPI-F3)
- Canaan K230 (CanMV-K230)

**MIPS:**
- MediaTek MT7620 (OpenWrt routers, Xiaomi Router 3G)

**LoongArch:**
- Loongson 3A5000, 3A6000, 2K1000LA

### Minimum Requirements

| Resource | Minimum | Recommended |
|----------|---------|-------------|
| RAM | 10MB free | 32MB+ free |
| Storage | 20MB | 50MB+ |
| CPU | 0.6GHz single core | Any |
| OS | Linux 3.x+ | Linux 5.x+ |

---

## Summary

### Voice Pipeline

**Strengths:**
- Clean interface-based design with 3 provider implementations
- Priority-based detection (explicit config > implicit fallback)
- Graceful error handling (media failures don't block message processing)
- Comprehensive test coverage

**Technical Debt:**
- `supportsAudioTranscription()` is too broad (whitelists entire protocols)
- Hardcoded model IDs in ElevenLabs (`scribe_v1`) and Groq (`whisper-large-v3`) transcripters

### Cross-Platform Binary

**Strengths:**
- CGO_ENABLED=0 for fully static binaries
- Comprehensive architecture support (8+ architectures)
- Platform-specific fixes in Makefile (MIPS ELF patching, Loong64 PTY)
- GoReleaser for automated releases with signing/notarization
- Well-documented hardware compatibility

**Implementation Notes:**
- macOS web launcher requires CGO_ENABLED=1 (unique exception)
- MIPS builds use softfloat for broader kernel compatibility
- The PATCH_MIPS_FLAGS and PTY_PATCH_LOONG64 workarounds indicate upstream library gaps
