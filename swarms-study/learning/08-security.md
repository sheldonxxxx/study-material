# Security: Swarms

## Security Model Overview

Swarms implements a privacy-first security model with the following core principles:
- **No telemetry collection** by default
- **Environment variable-based** secrets management
- **API key isolation** per service provider
- **No hardcoded credentials**

## Secrets Management

### Environment Variables

All sensitive configuration is managed via environment variables. The `.env.example` file documents required variables:

```bash
# Model Provider API Keys
OPENAI_API_KEY=""
ANTHROPIC_API_KEY=""
GEMINI_API_KEY=""
HUGGINGFACE_TOKEN=""
GROQ_API_KEY=""
PPLX_API_KEY=""
AI21_API_KEY=""

# Tool Provider API Keys
BING_BROWSER_API=""
BRAVESEARCH_API_KEY=""
TAVILY_API_KEY=""
YOU_API_KEY=""
EXA_API_KEY=""
MULTION_API_KEY=""

# Cloud Provider Configuration
AZURE_OPENAI_ENDPOINT=""
AZURE_OPENAI_DEPLOYMENT=""
AZURE_OPENAI_API_KEY=""
```

### Framework Configuration

```bash
SWARMS_API_KEY=""           # Swarms platform key
SWARMS_TELEMETRY_ON="false" # Telemetry disabled by default
SWARMS_VERBOSE_GLOBAL="False"
```

### .gitignore Coverage

The project includes comprehensive `.gitignore` to prevent accidental credential committed:
- `.env` files
- `agent_workspace/` directory
- Cache directories (`.mypy_cache`, `.ruff_cache`, etc.)

## Authentication Patterns

### API Key Authentication

The framework supports multiple authentication methods:
1. **Direct API Keys**: Environment variables for each provider
2. **Azure AD Tokens**: `AZURE_OPENAI_AD_TOKEN` for Azure authentication
3. **Bearer Tokens**: `HUGGINGFACE_TOKEN` for Hugging Face

### LiteLLM Integration

The `litellm` library (v1.76.1) handles authentication abstraction across providers, supporting:
- API key rotation
- Multiple provider configurations
- Unified interface with provider-specific auth

## Security Features Documented

| Feature | Description |
|---------|-------------|
| Environment Variables | Secure configuration management |
| No Telemetry | Privacy by design |
| Data Encryption | Protection of sensitive data |
| Access Control | Authorization mechanisms |
| Dependency Security | Dependency vulnerability management |
| Secure Installation | Integrity via package verification |

## Data Protection

### Workspace Isolation

Each agent run can specify a workspace directory:
```bash
WORKSPACE_DIR="agent_workspace"
```

This isolates agent artifacts and outputs from the host system.

### File Handling

The framework uses:
- `aiofiles` for async file operations with proper resource management
- `pypdf` for safe PDF processing without arbitrary code execution

## Vulnerability Reporting

The project maintains a security policy with clear reporting procedures:

1. **Report to**: kye@swarms.world
2. **Response time**: Acknowledgment within 24 hours
3. **Process**: Detailed assessment, security patch or advisory issued
4. **Scope**: Supported versions only

## Dependency Security

### Dependency Management

Poetry is used for reproducible dependency resolution:
```toml
[tool.poetry.dependencies]
python = ">=3.10,<4.0"
```

### Known Dependencies

Key dependencies and their security implications:

| Dependency | Purpose | Security Consideration |
|------------|---------|----------------------|
| `httpx` | HTTP client | Modern async HTTP with timeout support |
| `aiohttp` | Async HTTP | Connection pooling, timeouts |
| `pydantic` | Data validation | Input sanitization |
| `python-dotenv` | Env loading | Safe parsing of .env files |

## Operational Security

### Logging

Loguru is configured with sensible defaults:
- Structured logging without sensitive data leakage
- Environment-variable-based configuration
- No default credential printing

### Error Handling

Error messages are designed to:
- Not expose internal paths or configurations
- Use generic messages for API failures
- Log detailed errors server-side only

## Best Practices for Users

1. **Never commit `.env` files** - use environment variables or secrets managers
2. **Rotate API keys regularly** - especially for production deployments
3. **Use workspace isolation** - specify separate directories for agent outputs
4. **Enable telemetry only when needed** - `SWARMS_TELEMETRY_ON=false` by default
5. **Use virtual environments** - isolate dependencies per project
