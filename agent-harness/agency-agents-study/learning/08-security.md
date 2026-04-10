# Security Practices: agency-agents

## Security Philosophy

The agency-agents project addresses security through a dedicated Security Engineer agent and embedded security awareness across engineering agents. The approach follows defense-in-depth principles with proactive threat modeling, secure coding patterns, and automated scanning integrated into CI/CD pipelines.

## Security Engineer Agent

The Security Engineer agent (`engineering-security-engineer.md`) is a comprehensive application security specialist covering:

### Core Capabilities

- **Threat Modeling** — STRIDE analysis, risk prioritization, attack surface mapping
- **Vulnerability Assessment** — OWASP Top 10, CWE Top 25, severity classification
- **Secure Code Review** — Authentication, authorization, input validation, injection prevention
- **Security Architecture** — Zero-trust design, defense-in-depth, RBAC/ABAC

### Security Principles

```markdown
### Security-First Principles
- Never recommend disabling security controls as a solution
- Always assume user input is malicious — validate and sanitize everything at trust boundaries
- Prefer well-tested libraries over custom cryptographic implementations
- Treat secrets as first-class concerns — no hardcoded credentials, no secrets in logs
- Default to deny — whitelist over blacklist in access control and input validation
```

### Vulnerability Classification

| Severity | Examples | Response Time |
|----------|---------|---------------|
| Critical | Auth bypass, SQL injection, RCE | Under 48 hours |
| High | XSS, CSRF, IDOR | This sprint |
| Medium | Information disclosure, missing validation | Next sprint |
| Low | Missing security headers, verbose errors | Backlog |

## Threat Modeling Methodology

### STRIDE Analysis Framework

The Security Engineer agent uses STRIDE for threat modeling:

| Threat | Description | Mitigation |
|--------|-------------|------------|
| Spoofing | Impersonating users or components | MFA, token binding |
| Tampering | Modifying data or code | HMAC signatures, input validation |
| Repudiation | Denying actions taken | Immutable audit logging |
| Information Disclosure | Exposing sensitive data | Generic error responses, encryption |
| Denial of Service | Making system unavailable | Rate limiting, WAF |
| Elevation of Privilege | Gaining unauthorized access | RBAC, session isolation |

### Threat Model Document Template

```markdown
# Threat Model: [Application Name]

## System Overview
- **Architecture**: [Monolith/Microservices/Serverless]
- **Data Classification**: [PII, financial, health, public]
- **Trust Boundaries**: [User → API → Service → Database]

## STRIDE Analysis
| Threat | Component | Risk | Mitigation |
|--------|-----------|------|-----------|
| Spoofing | Auth endpoint | High | MFA + token binding |
| Tampering | API requests | High | HMAC signatures + input validation |

## Attack Surface
- External: Public APIs, OAuth flows, file uploads
- Internal: Service-to-service communication, message queues
- Data: Database queries, cache layers, log storage
```

## Secure Coding Patterns

### Input Validation (Python/FastAPI)

```python
from fastapi import FastAPI, Depends, HTTPException, status
from fastapi.security import HTTPBearer
from pydantic import BaseModel, Field, field_validator
import re

app = FastAPI()
security = HTTPBearer()

class UserInput(BaseModel):
    """Input validation with strict constraints."""
    username: str = Field(..., min_length=3, max_length=30)
    email: str = Field(..., max_length=254)

    @field_validator("username")
    @classmethod
    def validate_username(cls, v: str) -> str:
        if not re.match(r"^[a-zA-Z0-9_-]+$", v):
            raise ValueError("Username contains invalid characters")
        return v

    @field_validator("email")
    @classmethod
    def validate_email(cls, v: str) -> str:
        if not re.match(r"^[^@\s]+@[^@\s]+\.[^@\s]+$", v):
            raise ValueError("Invalid email format")
        return v

@app.post("/api/users")
async def create_user(
    user: UserInput,
    token: str = Depends(security)
):
    # 1. Authentication via dependency injection
    # 2. Input validated by Pydantic before handler
    # 3. Parameterized queries — never string concatenation
    # 4. Minimal data returned — no internal IDs or stack traces
    # 5. Security events logged for audit trail
    return {"status": "created", "username": user.username}
```

### Security Headers Configuration

```nginx
server {
    # Prevent MIME type sniffing
    add_header X-Content-Type-Options "nosniff" always;

    # Clickjacking protection
    add_header X-Frame-Options "DENY" always;

    # XSS filter (legacy browsers)
    add_header X-XSS-Protection "1; mode=block" always;

    # HSTS (1 year + subdomains)
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;

    # Content Security Policy
    add_header Content-Security-Policy "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; font-src 'self'; connect-src 'self'; frame-ancestors 'none'; base-uri 'self'; form-action 'self';" always;

    # Referrer Policy
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    # Permissions Policy
    add_header Permissions-Policy "camera=(), microphone=(), geolocation=(), payment=()" always;

    # Remove server version disclosure
    server_tokens off;
}
```

### Authentication & Authorization

The Security Engineer agent defines OAuth 2.0 / OIDC patterns:

**Zero-Trust Architecture Principles:**
- Never trust, always verify
- Least-privilege access
- Assume breach mentality
- Verify explicitly

**RBAC Implementation:**
- Role definitions with clear permission boundaries
- Session isolation for admin panels
- Token binding to prevent session hijacking

## Secrets Management

### Security Engineer Rules on Secrets

```markdown
### Critical Rules
- Treat secrets as first-class concerns
- No hardcoded credentials
- No secrets in logs
- Secrets rotation policies
- Encryption at rest and in transit
```

### Secrets Scanning in CI/CD

The project includes Gitleaks integration for secrets detection:

```yaml
secrets-scan:
  name: Secrets Detection
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v4
      with:
        fetch-depth: 0
    - name: Run Gitleaks
      uses: gitleaks/gitleaks-action@v2
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

## CI/CD Security Pipeline

### Complete Security Scanning Stage

```yaml
name: Security Scan

on:
  pull_request:
    branches: [main]

jobs:
  sast:
    name: Static Analysis (Semgrep)
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run Semgrep SAST
        uses: semgrep/semgrep-action@v1
        with:
          config: >-
            p/owasp-top-ten
            p/cwe-top-25

  dependency-scan:
    name: Dependency Audit (Trivy)
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          severity: 'CRITICAL,HIGH'
          exit-code: '1'

  secrets-scan:
    name: Secrets Detection (Gitleaks)
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - name: Run Gitleaks
        uses: gitleaks/gitleaks-action@v2
```

## Security Workflow Process

### Four-Phase Security Assessment

**Step 1: Reconnaissance & Threat Modeling**
- Map application architecture, data flows, trust boundaries
- Identify sensitive data (PII, credentials, financial data)
- Perform STRIDE analysis on each component
- Prioritize risks by likelihood and business impact

**Step 2: Security Assessment**
- Review code for OWASP Top 10 vulnerabilities
- Test authentication and authorization mechanisms
- Assess input validation and output encoding
- Evaluate secrets management and cryptographic implementations
- Check cloud/infrastructure security configuration

**Step 3: Remediation & Hardening**
- Provide prioritized findings with severity ratings
- Deliver concrete code-level fixes
- Implement security headers, CSP, transport security
- Set up automated scanning in CI/CD pipeline

**Step 4: Verification & Monitoring**
- Verify fixes resolve identified vulnerabilities
- Set up runtime security monitoring and alerting
- Establish security regression testing
- Create incident response playbooks

## Advanced Security Capabilities

### Cloud & Infrastructure Security

The Security Engineer agent covers:
- **CSPM** — Cloud security posture management across AWS, GCP, Azure
- **Container Security** — Falco, OPA runtime protection
- **IaC Security** — Terraform, CloudFormation review
- **Service Mesh** — Istio, Linkerd network security

### Incident Response & Forensics

- Security incident triage and root cause analysis
- Log analysis and attack pattern identification
- Post-incident remediation and hardening
- Breach impact assessment and containment

### Compliance Support

Frameworks addressed:
- PCI-DSS (payment card data)
- HIPAA (health information)
- SOC 2 (service organizations)
- GDPR (personal data)

## Security Success Metrics

The Security Engineer agent defines measurable outcomes:

- **Zero critical/high vulnerabilities reach production**
- **Mean time to remediate critical findings is under 48 hours**
- **100% of PRs pass automated security scanning before merge**
- **Security findings per release decrease quarter over quarter**
- **No secrets or credentials committed to version control**

## Security in Other Agents

### Code Reviewer Security Focus

The Code Reviewer agent prioritizes security during code review:

**🔴 Blockers (Must Fix):**
- Security vulnerabilities (injection, XSS, auth bypass)
- Input validation missing at trust boundaries
- Authentication/authorization flaws

**Review Checklist:**
- SQL injection vectors
- XSS attack surfaces
- CSRF protection
- Authentication bypass possibilities
- Authorization enforcement

### Frontend Developer Security Practices

The Frontend Developer agent includes security in Core Web Vitals:

**Security Deliverables:**
- Proper CSP implementation
- XSS prevention patterns
- Secure session management
- HTTPS-only transport
- Security headers

### Reality Checker Security Validation

The Reality Checker agent includes security in production readiness:

**Security Requirements:**
- Security headers validated
- No secrets in client-side code
- Input validation on forms
- HTTPS enforced
- Authentication flow tested

## Related Specialized Agents

### Blockchain Security Auditor

Specialized agent for smart contract security:
- Exploit analysis
- Vulnerability detection in EVM contracts
- Gas optimization security implications

### Threat Detection Engineer

Specialized detection agent:
- SIEM rules development
- Threat hunting methodologies
- ATT&CK framework mapping
- Detection pipeline development

### Compliance Auditor

Specialized compliance agent:
- SOC 2 preparation and auditing
- ISO 27001 compliance
- HIPAA requirements
- PCI-DSS validation

## Summary

Security in agency-agents is implemented through:

1. **Dedicated Security Engineer agent** with comprehensive threat modeling and secure coding patterns
2. **STRIDE methodology** for systematic threat analysis
3. **OWASP/CWE alignment** for vulnerability classification
4. **CI/CD security scanning** with SAST, SCA, and secrets detection
5. **Security-first principles** embedded across engineering agents
6. **Measurable security metrics** and compliance frameworks
7. **Defense-in-depth** architecture patterns
