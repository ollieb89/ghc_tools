---
description: 'Security audit workflow - systematic security review with OWASP Top 10, threat modeling, and findings report'
---

# Security Audit Prompt

Perform a comprehensive security audit of the target codebase, component, or pull request.

## Audit Workflow

### Step 1: Scope Definition

**Before starting, define:**
- Target: [file/directory/PR/entire codebase]
- Technology stack: [languages, frameworks, databases]
- Threat model: [internal users, public API, admin panel, etc.]
- Compliance requirements: [SOC2, PCI DSS, HIPAA, GDPR, none]

### Step 2: Attack Surface Identification

Map all entry points and data flows:

```
🔍 Entry Points Checklist:
□ API endpoints (REST, GraphQL, gRPC)
□ Web forms and user inputs
□ File upload handlers
□ Authentication endpoints (login, register, password reset)
□ WebSocket connections
□ Scheduled jobs and background tasks
□ CLI interfaces
□ Third-party integrations (webhooks, OAuth callbacks)
```

### Step 3: OWASP Top 10 Review

Systematically check for each vulnerability class:

| # | Vulnerability | Check For |
|---|--------------|-----------|
| A01 | Broken Access Control | Missing authorization checks, IDOR, privilege escalation |
| A02 | Cryptographic Failures | Weak algorithms, hardcoded secrets, insecure storage |
| A03 | Injection | SQLi, NoSQLi, command injection, XSS, LDAP injection |
| A04 | Insecure Design | Missing threat modeling, business logic flaws |
| A05 | Security Misconfiguration | Debug mode, default creds, verbose errors |
| A06 | Vulnerable Components | Outdated dependencies, known CVEs |
| A07 | Auth Failures | Weak passwords, missing MFA, session fixation |
| A08 | Data Integrity Failures | Insecure deserialization, unsigned updates |
| A09 | Logging Failures | Missing audit logs, log injection, PII in logs |
| A10 | SSRF | Unvalidated URL inputs, internal network access |

### Step 4: Code Pattern Analysis

Search for dangerous patterns:

```bash
# SQL Injection patterns
grep -rn "execute.*%s" --include="*.py"
grep -rn "f\".*SELECT.*{" --include="*.py"
grep -rn "query.*\+" --include="*.js"

# Command Injection patterns
grep -rn "subprocess\|os.system\|exec\|eval" --include="*.py"
grep -rn "child_process\|exec\|spawn" --include="*.js"

# XSS patterns
grep -rn "dangerouslySetInnerHTML\|innerHTML" --include="*.tsx"
grep -rn "v-html\|{{{" --include="*.vue"

# Hardcoded secrets
grep -rn "password.*=\|api_key.*=\|secret.*=" --include="*.py"
grep -rn "Bearer\|Basic " --include="*.ts"

# Insecure crypto
grep -rn "MD5\|SHA1\|DES\|RC4" --include="*.py"
```

### Step 5: Authentication & Authorization Review

```
🔐 Authentication Checklist:
□ Password hashing uses bcrypt/argon2/scrypt (not MD5/SHA1)
□ Rate limiting on login endpoints
□ Account lockout after failed attempts
□ Secure session management (httpOnly, secure, sameSite)
□ JWT tokens have expiration and proper validation
□ Password reset uses secure tokens with expiration

🛡️ Authorization Checklist:
□ All endpoints have explicit authorization checks
□ Role-based access control implemented
□ Resource ownership validated (no IDOR)
□ Admin functions protected
□ API keys scoped to minimum permissions
```

### Step 6: Data Protection Review

```
🗄️ Data Handling Checklist:
□ Sensitive data encrypted at rest
□ TLS 1.2+ for data in transit
□ PII minimization (collect only what's needed)
□ Data retention policies implemented
□ Secure deletion when required
□ Backup encryption
□ No sensitive data in logs
```

### Step 7: Dependency Analysis

```bash
# Python
pip-audit
safety check
pixi run pip-audit

# Node.js
npm audit
pnpm audit

# Check for known CVEs
gh api /repos/{owner}/{repo}/vulnerability-alerts
```

### Step 8: Findings Report

Generate a structured report:

```markdown
# Security Audit Report

**Target:** [component/repository]
**Date:** [YYYY-MM-DD]
**Auditor:** [agent/analyst]
**Scope:** [description]

## Executive Summary
- Critical findings: X
- High findings: X
- Medium findings: X
- Low findings: X

## Critical Findings

### [FINDING-001] [Title]
- **Severity:** Critical
- **CVSS:** X.X
- **Location:** `path/to/file.py:line`
- **Description:** [What the vulnerability is]
- **Impact:** [What an attacker could do]
- **Proof of Concept:** [How to exploit - be responsible]
- **Remediation:** [How to fix]
- **References:** [CVE, CWE, OWASP links]

## High Findings
[...]

## Medium Findings
[...]

## Low Findings
[...]

## Recommendations

### Immediate Actions (24-48 hours)
1. [Fix critical vulnerabilities]

### Short-term (1-2 weeks)
1. [Fix high/medium vulnerabilities]

### Long-term (1-3 months)
1. [Security architecture improvements]

## Appendix
- Tools used
- Scope limitations
- False positives
```

## Output Requirements

1. **Findings List**: Each finding with severity, location, and remediation
2. **Prioritized Remediation Plan**: Ordered by risk and effort
3. **Code Snippets**: Show vulnerable code and secure fix
4. **References**: Link to relevant CVEs, CWEs, OWASP documentation

## Example Usage

```
@prompt ghc_tools/prompts/security-audit.prompt.md

Target: apps/api/src/app/api/v1/endpoints/auth/
Stack: FastAPI, SQLAlchemy, PostgreSQL
Threat Model: Public API with user authentication
Compliance: SOC2
```

## Related Resources

- **Agent**: `ghc_tools/agents/security-expert.agent.md`
- **Instructions**: `ghc_tools/instructions/security-hardening.instructions.md`
- **Chatmode**: `ghc_tools/chatmodes/security-expert.chatmode.md`
