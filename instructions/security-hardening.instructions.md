---
description: 'Security hardening best practices for web applications - OWASP, injection prevention, authentication, encryption'
applyTo: '**/*.py, **/*.js, **/*.ts, **/*.java, **/*.go, **/*.rb'
subjects: [security, owasp, vulnerability, injection, xss, authentication, encryption, hardening]
category: security
---

# Security Hardening Instructions

Apply these security best practices to all application code.

## 1. Injection Prevention

### SQL Injection

```python
# ❌ BAD: String concatenation (SQLi vulnerable)
query = f"SELECT * FROM users WHERE email = '{email}'"
cursor.execute(query)

# ✅ GOOD: Parameterized queries
cursor.execute("SELECT * FROM users WHERE email = %s", (email,))

# ✅ GOOD: ORM with proper filtering
User.objects.filter(email=email).first()
```

```javascript
// ❌ BAD: Template literals in SQL
const query = `SELECT * FROM users WHERE id = ${userId}`;

// ✅ GOOD: Parameterized query
const query = 'SELECT * FROM users WHERE id = $1';
await pool.query(query, [userId]);
```

### Command Injection

```python
# ❌ BAD: Shell injection vulnerable
os.system(f"ping {user_input}")
subprocess.call(f"convert {filename}", shell=True)

# ✅ GOOD: Avoid shell, use list arguments
subprocess.run(["ping", "-c", "4", validated_host], shell=False)

# ✅ GOOD: Whitelist allowed values
ALLOWED_HOSTS = {"google.com", "github.com"}
if host in ALLOWED_HOSTS:
    subprocess.run(["ping", "-c", "4", host])
```

### NoSQL Injection

```javascript
// ❌ BAD: Operator injection in MongoDB
db.users.find({ username: req.body.username });
// Attacker sends: { "username": { "$ne": "" } }

// ✅ GOOD: Sanitize and type-check input
const username = String(req.body.username).slice(0, 100);
db.users.find({ username: username });
```

## 2. Cross-Site Scripting (XSS) Prevention

### Output Encoding

```python
# ❌ BAD: Unescaped output
return f"<h1>Welcome, {username}</h1>"

# ✅ GOOD: HTML escape
from markupsafe import escape
return f"<h1>Welcome, {escape(username)}</h1>"

# ✅ GOOD: Use templating engine auto-escape
# Jinja2 with autoescape=True (default in Flask)
```

```javascript
// ❌ BAD: innerHTML with user data
element.innerHTML = userContent;

// ✅ GOOD: textContent for plain text
element.textContent = userContent;

// ✅ GOOD: DOMPurify for rich content
import DOMPurify from 'dompurify';
element.innerHTML = DOMPurify.sanitize(userContent);
```

### Content Security Policy

```python
# Add CSP header to prevent inline scripts
response.headers['Content-Security-Policy'] = (
    "default-src 'self'; "
    "script-src 'self' 'nonce-{nonce}'; "
    "style-src 'self' 'unsafe-inline'; "
    "img-src 'self' data: https:; "
    "connect-src 'self' https://api.example.com"
)
```

## 3. Authentication Security

### Password Handling

```python
# ❌ BAD: Plain text or weak hashing
password_hash = hashlib.md5(password.encode()).hexdigest()

# ✅ GOOD: bcrypt with sufficient work factor
import bcrypt
password_hash = bcrypt.hashpw(password.encode(), bcrypt.gensalt(rounds=12))

# ✅ GOOD: Verify with constant-time comparison
if bcrypt.checkpw(password.encode(), stored_hash):
    # authenticated
```

### Session Security

```python
# ✅ GOOD: Secure session configuration
app.config.update(
    SESSION_COOKIE_SECURE=True,      # HTTPS only
    SESSION_COOKIE_HTTPONLY=True,    # No JavaScript access
    SESSION_COOKIE_SAMESITE='Lax',   # CSRF protection
    PERMANENT_SESSION_LIFETIME=3600,  # 1 hour timeout
)

# ✅ GOOD: Regenerate session on auth change
session.regenerate()  # After login/logout/privilege change
```

### JWT Security

```python
# ❌ BAD: No algorithm verification
jwt.decode(token, secret, algorithms=["HS256", "none"])

# ✅ GOOD: Explicit algorithm, verify signature
jwt.decode(
    token,
    secret,
    algorithms=["HS256"],  # Only allow expected algorithm
    options={"require": ["exp", "iat", "sub"]}  # Require claims
)

# ✅ GOOD: Short expiration, refresh tokens
payload = {
    "sub": user_id,
    "exp": datetime.utcnow() + timedelta(minutes=15),  # Short-lived
    "iat": datetime.utcnow(),
}
```

## 4. Authorization

### Access Control

```python
# ❌ BAD: No authorization check
@app.route("/api/documents/<doc_id>")
def get_document(doc_id):
    return Document.query.get(doc_id)

# ✅ GOOD: Verify ownership/permissions
@app.route("/api/documents/<doc_id>")
@login_required
def get_document(doc_id):
    document = Document.query.get_or_404(doc_id)
    if document.owner_id != current_user.id:
        abort(403)  # Forbidden
    return document
```

### IDOR Prevention

```python
# ❌ BAD: Direct object reference
user = User.query.get(request.args["user_id"])

# ✅ GOOD: Scope queries to current user's access
user = User.query.filter_by(
    id=request.args["user_id"],
    organization_id=current_user.organization_id  # Scope to org
).first_or_404()
```

## 5. Cryptography

### Encryption

```python
# ❌ BAD: ECB mode, weak algorithms
from Crypto.Cipher import DES
cipher = DES.new(key, DES.MODE_ECB)

# ✅ GOOD: AES-GCM (authenticated encryption)
from cryptography.hazmat.primitives.ciphers.aead import AESGCM
key = AESGCM.generate_key(bit_length=256)
aesgcm = AESGCM(key)
nonce = os.urandom(12)  # Unique per encryption
ciphertext = aesgcm.encrypt(nonce, plaintext, associated_data)
```

### Secrets Management

```python
# ❌ BAD: Hardcoded secrets
API_KEY = "sk-1234567890abcdef"
DATABASE_URL = "postgresql://user:password123@localhost/db"

# ✅ GOOD: Environment variables
import os
API_KEY = os.environ["API_KEY"]  # Raises if missing

# ✅ GOOD: Secret manager for production
from google.cloud import secretmanager
client = secretmanager.SecretManagerServiceClient()
secret = client.access_secret_version(name="projects/x/secrets/api-key/versions/latest")
```

### Random Number Generation

```python
# ❌ BAD: Predictable random (not cryptographic)
import random
token = ''.join(random.choices(string.ascii_letters, k=32))

# ✅ GOOD: Cryptographically secure random
import secrets
token = secrets.token_urlsafe(32)  # 256 bits of entropy
```

## 6. Security Headers

```python
# Add to all responses
@app.after_request
def add_security_headers(response):
    response.headers['X-Content-Type-Options'] = 'nosniff'
    response.headers['X-Frame-Options'] = 'DENY'
    response.headers['X-XSS-Protection'] = '1; mode=block'
    response.headers['Strict-Transport-Security'] = 'max-age=31536000; includeSubDomains'
    response.headers['Referrer-Policy'] = 'strict-origin-when-cross-origin'
    response.headers['Permissions-Policy'] = 'geolocation=(), microphone=()'
    return response
```

## 7. Input Validation

```python
# ✅ GOOD: Whitelist validation with Pydantic
from pydantic import BaseModel, EmailStr, constr, validator

class UserCreate(BaseModel):
    email: EmailStr
    username: constr(min_length=3, max_length=30, regex=r'^[a-zA-Z0-9_]+$')
    age: int = Field(ge=13, le=120)
    
    @validator('username')
    def username_not_reserved(cls, v):
        reserved = {'admin', 'root', 'system'}
        if v.lower() in reserved:
            raise ValueError('Reserved username')
        return v
```

## 8. Logging Security

```python
# ❌ BAD: Logging sensitive data
logger.info(f"User login: {email}, password: {password}")

# ✅ GOOD: Sanitize logs
logger.info(f"User login attempt: {email}")  # No password

# ✅ GOOD: Mask sensitive fields
def mask_sensitive(data: dict) -> dict:
    sensitive_keys = {'password', 'token', 'secret', 'api_key', 'ssn'}
    return {
        k: '***REDACTED***' if k.lower() in sensitive_keys else v
        for k, v in data.items()
    }
```

## 9. Dependency Security

```bash
# Scan for vulnerabilities
pip-audit                    # Python
npm audit                    # Node.js
snyk test                    # Multi-language

# Pin dependencies to exact versions
pip freeze > requirements.txt
npm shrinkwrap

# Use Dependabot or Renovate for automated updates
```

## 10. Error Handling

```python
# ❌ BAD: Exposing stack traces
@app.errorhandler(Exception)
def handle_error(e):
    return str(e), 500  # Leaks internal details

# ✅ GOOD: Generic error messages in production
@app.errorhandler(Exception)
def handle_error(e):
    app.logger.exception("Unhandled exception")  # Log full details
    if app.debug:
        raise e
    return {"error": "An internal error occurred"}, 500
```

## Quick Reference: Security Checklist

| Category | Check |
|----------|-------|
| **Injection** | Parameterized queries, no shell=True |
| **XSS** | Output encoding, CSP headers |
| **Auth** | bcrypt passwords, secure sessions |
| **AuthZ** | Ownership checks, scoped queries |
| **Crypto** | AES-GCM, secrets.token_urlsafe |
| **Secrets** | Environment variables, no hardcoding |
| **Headers** | HSTS, X-Frame-Options, CSP |
| **Logging** | No sensitive data in logs |
| **Deps** | Regular vulnerability scanning |
| **Errors** | Generic messages in production |
