---
description: 'Security vulnerability remediation workflow - root cause analysis, secure fix implementation, and verification'
---

# Security Fix Prompt

Remediate a specific security vulnerability with a structured approach.

## Input Requirements

Provide the following:
- **Vulnerability Type**: [SQL injection, XSS, IDOR, etc.]
- **CVE/CWE**: [if applicable]
- **Affected Code**: [file path and relevant code]
- **Severity**: [Critical/High/Medium/Low]
- **Context**: [How the vulnerability was discovered]

## Remediation Workflow

### Step 1: Understand the Vulnerability

**Analyze the root cause:**

```
📋 Vulnerability Analysis:
- What: [Description of the vulnerability]
- Where: [Exact location in code]
- How: [Attack vector - how could it be exploited]
- Why: [Root cause - why does this code allow the attack]
- Impact: [What an attacker could achieve]
```

**Example Analysis:**

```python
# VULNERABLE CODE
# File: apps/api/src/app/api/v1/endpoints/users.py:45
@router.get("/users/{user_id}")
async def get_user(user_id: str, db: AsyncSession = Depends(get_db)):
    # SQL Injection vulnerability - user_id directly interpolated
    query = f"SELECT * FROM users WHERE id = '{user_id}'"
    result = await db.execute(text(query))
    return result.fetchone()

# Root Cause: String interpolation allows attacker-controlled input
# into SQL query without sanitization or parameterization.
# Attack: user_id = "' OR '1'='1" returns all users
```

### Step 2: Design the Secure Fix

**Apply defense-in-depth:**

1. **Input Validation**: Validate type, format, range
2. **Parameterized Queries**: Never interpolate user input
3. **Output Encoding**: Encode data for the output context
4. **Least Privilege**: Minimize database permissions

**Secure Fix Example:**

```python
# SECURE CODE
from pydantic import conint
from sqlalchemy import select

@router.get("/users/{user_id}")
async def get_user(
    user_id: conint(gt=0),  # Input validation: positive integer only
    db: AsyncSession = Depends(get_db),
    current_user: User = Depends(get_current_user)  # Authentication
):
    # Authorization check - can this user access this resource?
    if not await can_view_user(current_user, user_id):
        raise HTTPException(status_code=403, detail="Access denied")
    
    # Parameterized query - user_id is bound, not interpolated
    stmt = select(User).where(User.id == user_id)
    result = await db.execute(stmt)
    user = result.scalar_one_or_none()
    
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    
    return UserResponse.model_validate(user)  # Explicit response model
```

### Step 3: Implementation Checklist

Apply the fix with these safeguards:

```
✅ Fix Checklist:
□ Root cause addressed (not just symptoms)
□ Input validation added
□ Parameterized queries used
□ Authorization checks in place
□ Error handling doesn't leak information
□ Logging added for security events
□ Type hints and response models defined
□ Backward compatibility maintained (or migration path)
```

### Step 4: Write Security Tests

**Test the fix AND the vulnerability:**

```python
# tests/security/test_user_endpoint_security.py
import pytest
from httpx import AsyncClient

class TestUserEndpointSecurity:
    """Security tests for user endpoint after vulnerability fix."""
    
    @pytest.mark.parametrize("malicious_input", [
        "' OR '1'='1",           # SQL injection
        "1; DROP TABLE users;",  # SQL injection with command
        "-1",                    # Negative ID
        "abc",                   # Non-numeric
        "1' AND 1=1--",          # Boolean-based SQLi
        "1 UNION SELECT * FROM users--",  # UNION-based SQLi
    ])
    async def test_sql_injection_blocked(
        self, client: AsyncClient, malicious_input: str
    ):
        """Verify SQL injection attempts are rejected."""
        response = await client.get(f"/api/v1/users/{malicious_input}")
        # Should return 422 (validation error) or 400, NOT 200
        assert response.status_code in [400, 422]
        
    async def test_idor_prevented(
        self, client: AsyncClient, 
        user_token: str,
        other_user_id: int
    ):
        """Verify users cannot access other users' data."""
        response = await client.get(
            f"/api/v1/users/{other_user_id}",
            headers={"Authorization": f"Bearer {user_token}"}
        )
        assert response.status_code == 403
        
    async def test_valid_request_succeeds(
        self, client: AsyncClient,
        user_token: str,
        user_id: int
    ):
        """Verify legitimate requests still work."""
        response = await client.get(
            f"/api/v1/users/{user_id}",
            headers={"Authorization": f"Bearer {user_token}"}
        )
        assert response.status_code == 200
        assert response.json()["id"] == user_id
```

### Step 5: Verification

**Confirm the fix is effective:**

```
🔍 Verification Steps:
1. Run unit tests: pytest tests/security/
2. Run integration tests with malicious inputs
3. Manual testing with security tools (if applicable)
4. Code review by second pair of eyes
5. Static analysis scan (bandit, semgrep)
6. Dynamic testing if exposed endpoint
```

**Verification Commands:**

```bash
# Python static analysis
bandit -r apps/api/src/

# Run security tests
pytest tests/security/ -v

# Check for regressions
pytest tests/ -v --tb=short

# Semgrep for security patterns
semgrep --config=p/python-security apps/api/
```

### Step 6: Documentation

**Document the fix:**

```markdown
## Security Fix: SQL Injection in User Endpoint

**CVE/CWE:** CWE-89 (SQL Injection)
**Severity:** Critical
**Fixed in:** PR #123
**Date:** YYYY-MM-DD

### Summary
Fixed SQL injection vulnerability in `/api/v1/users/{user_id}` endpoint
that allowed attackers to extract or modify database contents.

### Changes
- Replaced string interpolation with parameterized SQLAlchemy query
- Added input validation (positive integer constraint)
- Added authorization check to prevent IDOR
- Added security tests for common injection patterns

### Testing
- Unit tests: `tests/security/test_user_endpoint_security.py`
- Manual verification with SQLMap: No vulnerabilities detected

### References
- OWASP SQL Injection: https://owasp.org/www-community/attacks/SQL_Injection
- CWE-89: https://cwe.mitre.org/data/definitions/89.html
```

## Quick Reference: Common Fixes

### SQL Injection
```python
# ❌ Vulnerable
query = f"SELECT * FROM users WHERE id = '{user_id}'"

# ✅ Secure (SQLAlchemy ORM)
stmt = select(User).where(User.id == user_id)

# ✅ Secure (parameterized raw SQL)
stmt = text("SELECT * FROM users WHERE id = :id")
result = await db.execute(stmt, {"id": user_id})
```

### XSS
```tsx
// ❌ Vulnerable
<div dangerouslySetInnerHTML={{__html: userContent}} />

// ✅ Secure
<div>{userContent}</div>  // React auto-escapes

// ✅ If HTML needed, sanitize first
import DOMPurify from 'dompurify';
<div dangerouslySetInnerHTML={{__html: DOMPurify.sanitize(userContent)}} />
```

### Command Injection
```python
# ❌ Vulnerable
os.system(f"convert {user_filename} output.png")

# ✅ Secure
import shlex
subprocess.run(["convert", user_filename, "output.png"], check=True)
```

### IDOR
```python
# ❌ Vulnerable - no ownership check
@router.get("/orders/{order_id}")
async def get_order(order_id: int, db: Session):
    return db.get(Order, order_id)

# ✅ Secure - verify ownership
@router.get("/orders/{order_id}")
async def get_order(
    order_id: int,
    db: Session,
    current_user: User = Depends(get_current_user)
):
    order = db.get(Order, order_id)
    if order.user_id != current_user.id:
        raise HTTPException(403, "Access denied")
    return order
```

## Related Resources

- **Audit Prompt**: `ghc_tools/prompts/security-audit.prompt.md`
- **Agent**: `ghc_tools/agents/security-expert.agent.md`
- **Instructions**: `ghc_tools/instructions/security-hardening.instructions.md`
