# CI/CD - Hermes Agent

## Workflow Overview

The project uses **GitHub Actions** for CI/CD with 4 primary workflows.

## Workflows

### 1. Tests (`tests.yml`)

**Trigger**: Push to `main`, Pull Requests to `main`

**Jobs**:
- `test` on `ubuntu-latest`, 10-minute timeout

**Steps**:
```bash
# Checkout
actions/checkout@v4

# Install uv
astral-sh/setup-uv@v5

# Set Python 3.11
uv python install 3.11

# Create venv and install
uv venv .venv --python 3.11
uv pip install -e ".[all,dev]"

# Run tests (skip integration tests)
python -m pytest tests/ -q --ignore=tests/integration --tb=short -n auto
```

**Environment Variables** (security):
- `OPENROUTER_API_KEY: ""` (empty, ensures no real API calls)
- `OPENAI_API_KEY: ""`
- `NOUS_API_KEY: ""`

**Concurrency**: Cancels in-progress runs for same PR/branch

---

### 2. Docs Site Checks (`docs-site-checks.yml`)

**Trigger**: PRs modifying `website/**` or the workflow itself, or manual `workflow_dispatch`

**Jobs**:
- `docs-site-checks` on `ubuntu-latest`

**Steps**:
```bash
# Checkout
actions/checkout@v4

# Node 20 setup with npm cache
actions/setup-node@v4 (node-version: 20, cache: npm)

# Install website deps
npm ci (working-directory: website)

# Python 3.11
actions/setup-python@v5 (python-version: '3.11')

# Install ascii-guard
python -m pip install ascii-guard

# Lint diagrams
npm run lint:diagrams (working-directory: website)

# Build Docusaurus
npm run build (working-directory: website)
```

---

### 3. Deploy Site (`deploy-site.yml`)

**Trigger**: Push to `main` with changes to `website/**`, `landingpage/**`, or this workflow; or manual `workflow_dispatch`

**Environment**: `github-pages`

**Steps**:
```bash
# Checkout
actions/checkout@v4

# Node 20 setup
actions/setup-node@v4 (node-version: 20)

# Install deps
npm ci (working-directory: website)

# Build Docusaurus
npm run build (working-directory: website)

# Stage deployment
mkdir -p _site/docs
cp -r landingpage/* _site/
cp -r website/build/* _site/docs/
echo "hermes-agent.nousresearch.com" > _site/CNAME

# Upload artifact
actions/upload-pages-artifact@v3 (path: _site)

# Deploy to GitHub Pages
actions/deploy-pages@v4
```

**Concurrency**: `pages` group, does NOT cancel in-progress

---

### 4. Supply Chain Audit (`supply-chain-audit.yml`)

**Trigger**: PR opened, synchronized, or reopened

**Permissions**: `pull-requests: write`, `contents: read`

**Purpose**: Scans PR diff for supply chain attack patterns

**Scanned Patterns**:
| Pattern | Severity | Description |
|---------|----------|-------------|
| `.pth` files | CRITICAL | Auto-execute on Python startup |
| `base64 + exec/eval` | CRITICAL | Litellm attack pattern |
| `subprocess` with encoded args | CRITICAL | Payload execution |
| `base64` encoding alone | WARNING | Obfuscation indicator |
| `exec/eval` with string args | WARNING | Dynamic code execution |
| `POST/PUT` network calls | WARNING | Potential exfiltration |
| `setup.py`, `__init__.pth` | WARNING | Install hook files |
| `marshal/pickle/compile` | WARNING | Code object injection |

**Behavior**:
- Posts PR comment with findings
- CRITICAL findings fail the workflow with error
- Non-critical findings post warning comment for maintainer review

---

### 5. Nix (`nix.yml`)

**Trigger**: Push to `main` or PR with changes to `flake.nix`, `flake.lock`, `nix/**`, `pyproject.toml`, `uv.lock`, key Python files

**Matrix**: `["ubuntu-latest", "macos-latest"]`

**Linux Steps**:
```bash
actions/checkout@v4
DeterminateSystems/nix-installer-action@main
DeterminateSystems/magic-nix-cache-action@main
nix flake check --print-build-logs  # Only on Linux
nix build --print-build-logs          # Only on Linux
```

**macOS Steps**:
```bash
nix flake show --json > /dev/null  # Evaluate only
```

**Concurrency**: `nix-${{ github.ref }}`, cancels in-progress

**Timeout**: 30 minutes

---

## Nix Flake Configuration

```nix
{
  description = "Hermes Agent - AI agent framework by Nous Research";

  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-24.11";
    flake-parts = { ... };
    pyproject-nix = { ... };
    uv2nix = { ... };
    pyproject-build-systems = { ... };
  };

  outputs = inputs: inputs.flake-parts.lib.mkFlake { inherit inputs; } {
    systems = [ "x86_64-linux" "aarch64-linux" "aarch64-darwin" ];
    imports = [ ./nix/packages.nix ./nix/nixosModules.nix ./nix/checks.nix ./nix/devShell.nix ];
  };
}
```

## Installation Scripts

### Shell Installer (`scripts/install.sh`)
- Linux/macOS/WSL2
- Installs Python, Node.js, dependencies, and `hermes` command
- No prerequisites except git

### PowerShell Installer (`scripts/install.ps1`)
- Windows (PowerShell)

## Release Process

- Release notes in `RELEASE_v*.md` at repo root
- Version in `pyproject.toml` (`project.version`)
- Python package uses setuptools with `pyproject.toml`

## Dependency Management

- **Production**: `pyproject.toml` dependencies with version bounds
- **Development**: `uv` for fast installs, pytest for testing
- **Nix**: `flake.lock` for reproducible builds
- **Supply Chain**: Audit workflow scans for malicious patterns

## Security Considerations

1. API keys excluded from test runs
2. Supply chain audit on every PR
3. `.pth` file detection (Python startup code execution)
4. Dangerous command pattern detection
5. No secrets logged in CI output
