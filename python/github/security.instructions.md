---
description: "Application security guidelines and best practices for Python projects"
applyTo: "**/*.py,**/*.pyi,**/*.toml,**/*.yml,**/*.yaml"
---

# Application Security Guidelines

## High-Level Principles

- Security is not optional — it must be embedded in every phase of development, not added as an afterthought.
- Adopt a **defense-in-depth** strategy: multiple layers of protection so that a failure in one does not compromise the system.
- Apply the **principle of least privilege**: code, services, and users should have only the minimum permissions needed.
- Assume all external input is hostile until validated.
- Keep the attack surface minimal: disable unused features, remove dead code, limit exposed endpoints.

---

## 1. Input Validation & Sanitization

### Rules

- **Validate all external input** at the boundary (API endpoints, CLI arguments, file uploads, environment variables, form data).
- Use **allowlists** (whitelist) over denylists (blacklist) for validation.
- Use **Pydantic models** for structured input validation with strict type coercion.
- Validate data types, ranges, lengths, formats, and allowed characters.
- Never trust client-side validation alone.

### Patterns

```python
from pydantic import BaseModel, Field, field_validator
import re

class UserRegistration(BaseModel):
    """Validated user registration input."""
    username: str = Field(min_length=3, max_length=30, pattern=r"^[a-zA-Z0-9_]+$")
    email: str = Field(max_length=254)
    age: int = Field(ge=13, le=150)

    @field_validator("email")
    @classmethod
    def validate_email(cls, v: str) -> str:
        if not re.match(r"^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$", v):
            raise ValueError("Invalid email format")
        return v.lower()
```

### Path Traversal Prevention

```python
from pathlib import Path

def safe_read_file(base_dir: Path, user_filename: str) -> str:
    """Read a file safely, preventing path traversal attacks."""
    # Resolve to absolute path and verify it's within the allowed directory
    safe_path = (base_dir / user_filename).resolve()
    if not safe_path.is_relative_to(base_dir.resolve()):
        raise ValueError("Access denied: path traversal detected")
    return safe_path.read_text()
```

### File Upload Validation

```python
ALLOWED_EXTENSIONS = {".jpg", ".jpeg", ".png", ".pdf"}
MAX_FILE_SIZE = 10 * 1024 * 1024  # 10 MB

def validate_upload(filename: str, file_size: int, content_type: str) -> None:
    """Validate an uploaded file against security constraints."""
    ext = Path(filename).suffix.lower()
    if ext not in ALLOWED_EXTENSIONS:
        raise ValueError(f"File extension '{ext}' is not allowed")
    if file_size > MAX_FILE_SIZE:
        raise ValueError(f"File exceeds maximum size of {MAX_FILE_SIZE} bytes")
    # Never rely solely on content_type from the client
    # Use python-magic or similar for server-side MIME detection
```

---

## 2. Authentication & Authorization

### Rules

- Never implement custom authentication schemes — use proven libraries and protocols (OAuth 2.0, OpenID Connect).
- Use **bcrypt**, **argon2**, or **scrypt** for password hashing — never MD5, SHA-1, or plain SHA-256.
- Enforce strong password policies (minimum length, complexity) at the application level.
- Implement account lockout or rate limiting after failed login attempts.
- Use short-lived tokens (JWT, session tokens) with proper expiration and rotation.
- Always verify authorization on the server side for every request, not just at the UI level.

### Password Hashing

```python
import argon2

password_hasher = argon2.PasswordHasher(
    time_cost=3,
    memory_cost=65536,
    parallelism=4,
)

def hash_password(password: str) -> str:
    """Hash a password using Argon2id."""
    return password_hasher.hash(password)

def verify_password(stored_hash: str, password: str) -> bool:
    """Verify a password against a stored hash."""
    try:
        return password_hasher.verify(stored_hash, password)
    except argon2.exceptions.VerifyMismatchError:
        return False
```

### JWT Token Handling

```python
import jwt
from datetime import datetime, timedelta, UTC

SECRET_KEY: str  # Load from environment, never hardcode

def create_access_token(user_id: str, *, expires_in: timedelta = timedelta(minutes=15)) -> str:
    """Create a short-lived JWT access token."""
    payload = {
        "sub": user_id,
        "iat": datetime.now(UTC),
        "exp": datetime.now(UTC) + expires_in,
        "type": "access",
    }
    return jwt.encode(payload, SECRET_KEY, algorithm="HS256")

def decode_token(token: str) -> dict:
    """Decode and validate a JWT token."""
    try:
        return jwt.decode(token, SECRET_KEY, algorithms=["HS256"])
    except jwt.ExpiredSignatureError:
        raise AuthenticationError("Token has expired")
    except jwt.InvalidTokenError:
        raise AuthenticationError("Invalid token")
```

### Role-Based Access Control (RBAC)

```python
from enum import StrEnum
from functools import wraps
from typing import Callable

class Role(StrEnum):
    VIEWER = "viewer"
    EDITOR = "editor"
    ADMIN = "admin"

ROLE_HIERARCHY: dict[Role, int] = {
    Role.VIEWER: 1,
    Role.EDITOR: 2,
    Role.ADMIN: 3,
}

def require_role(minimum_role: Role) -> Callable:
    """Decorator to enforce minimum role requirement."""
    def decorator(func: Callable) -> Callable:
        @wraps(func)
        def wrapper(*args, current_user: User, **kwargs):
            if ROLE_HIERARCHY[current_user.role] < ROLE_HIERARCHY[minimum_role]:
                raise PermissionError(f"Requires at least '{minimum_role}' role")
            return func(*args, current_user=current_user, **kwargs)
        return wrapper
    return decorator
```

---

## 3. Secrets Management

### Rules

- **Never** hardcode secrets, API keys, tokens, passwords, or connection strings in source code.
- **Never** commit secrets to version control — even in private repositories.
- Use **environment variables** for runtime secrets.
- Use a **secrets manager** (HashiCorp Vault, AWS Secrets Manager, Azure Key Vault, 1Password) for production systems.
- Rotate secrets regularly and on suspected compromise.
- Use `.env` files only for local development; add `.env` to `.gitignore`.

### Pre-Commit Secret Detection

```yaml
# .pre-commit-config.yaml (add to existing hooks)
  - repo: https://github.com/Yelp/detect-secrets
    rev: v1.4.0
    hooks:
      - id: detect-secrets
        args: ['--baseline', '.secrets.baseline']
```

### Safe Configuration Pattern

```python
from pydantic_settings import BaseSettings
from pydantic import SecretStr

class AppSettings(BaseSettings):
    """Application settings with secret handling."""
    database_url: SecretStr          # SecretStr prevents accidental logging
    api_key: SecretStr
    jwt_secret: SecretStr
    debug: bool = False

    model_config = {
        "env_prefix": "APP_",
        "env_file": ".env",
        "env_file_encoding": "utf-8",
    }

# Usage: settings.database_url.get_secret_value()
```

---

## 4. SQL Injection & Database Security

### Rules

- **Never** construct SQL queries with string formatting, f-strings, or concatenation.
- Always use **parameterized queries** or an ORM with proper binding.
- Use the principle of least privilege for database credentials (read-only where possible).
- Validate and sanitize all user input before passing to queries.

### Patterns

```python
# DANGEROUS — SQL injection vulnerability
query = f"SELECT * FROM users WHERE email = '{user_email}'"  # NEVER DO THIS

# SAFE — parameterized query
cursor.execute("SELECT * FROM users WHERE email = %s", (user_email,))

# SAFE — SQLAlchemy ORM
user = session.query(User).filter(User.email == user_email).first()

# SAFE — SQLAlchemy Core with bound parameters
stmt = select(users_table).where(users_table.c.email == bindparam("email"))
result = connection.execute(stmt, {"email": user_email})
```

---

## 5. Cross-Site Scripting (XSS) Prevention

### Rules

- **Always** escape user-provided content before rendering in HTML.
- Use template engines with **auto-escaping enabled** (Jinja2: `autoescape=True`).
- Set the `Content-Type` header correctly for all responses.
- Use Content Security Policy (CSP) headers to restrict script sources.

### Patterns

```python
from markupsafe import escape

# Always escape user input in HTML contexts
safe_username = escape(user_input)

# Jinja2 with auto-escaping (default in Flask/FastAPI)
from jinja2 import Environment, select_autoescape

env = Environment(autoescape=select_autoescape(["html", "xml"]))
```

---

## 6. Cross-Site Request Forgery (CSRF) Protection

### Rules

- Use **CSRF tokens** for all state-changing operations (POST, PUT, DELETE).
- Validate the `Origin` and `Referer` headers on the server.
- Use `SameSite` attribute on cookies (`Lax` or `Strict`).
- For APIs, prefer token-based authentication (Bearer tokens) over cookie-based sessions.

---

## 7. HTTP Security Headers

### Required Headers

```python
# FastAPI / Starlette middleware example
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request
from starlette.responses import Response

class SecurityHeadersMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next) -> Response:
        response = await call_next(request)
        response.headers["X-Content-Type-Options"] = "nosniff"
        response.headers["X-Frame-Options"] = "DENY"
        response.headers["X-XSS-Protection"] = "0"  # Disabled; use CSP instead
        response.headers["Strict-Transport-Security"] = "max-age=31536000; includeSubDomains"
        response.headers["Content-Security-Policy"] = "default-src 'self'"
        response.headers["Referrer-Policy"] = "strict-origin-when-cross-origin"
        response.headers["Permissions-Policy"] = "camera=(), microphone=(), geolocation=()"
        return response
```

---

## 8. Dependency Security

### Rules

- Audit dependencies for known vulnerabilities regularly.
- Pin dependencies to exact versions in lock files for reproducible builds.
- Remove unused dependencies.
- Prefer well-maintained packages with active security response.

### Tools & Automation

```bash
# Audit dependencies for vulnerabilities
pip audit
uv pip audit

# GitHub Dependabot — enable in .github/dependabot.yml
```

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: pip
    directory: "/"
    schedule:
      interval: weekly
    open-pull-requests-limit: 10
    labels:
      - "dependencies"
      - "security"
```

### CI Security Scanning

```yaml
# Add to .github/workflows/ci.yml
  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v4
      - run: uv sync --group dev
      - run: uv run pip audit
      - name: Run Bandit security linter
        run: uv run bandit -r src/ -c pyproject.toml
      - name: Run Semgrep
        uses: semgrep/semgrep-action@v1
        with:
          config: p/python
```

---

## 9. Cryptography

### Rules

- Use the `secrets` module for generating tokens, API keys, and nonces — never `random`.
- Use the `cryptography` library (or `PyNaCl`) for encryption — never roll your own.
- Use AES-256-GCM for symmetric encryption; RSA-OAEP or ECDH for asymmetric.
- Always use authenticated encryption (GCM, ChaCha20-Poly1305).
- Store encryption keys separately from encrypted data.

### Patterns

```python
import secrets
from cryptography.fernet import Fernet

# Generate cryptographically secure tokens
api_key = secrets.token_urlsafe(32)
session_id = secrets.token_hex(32)

# Symmetric encryption with Fernet (AES-128-CBC + HMAC)
key = Fernet.generate_key()  # Store securely, never in code
cipher = Fernet(key)
encrypted = cipher.encrypt(b"sensitive data")
decrypted = cipher.decrypt(encrypted)
```

---

## 10. Logging & Monitoring for Security

### Rules

- Log all authentication events (login, logout, failed attempts, password changes).
- Log all authorization failures (access denied events).
- Log all input validation failures from external sources.
- **Never** log sensitive data: passwords, tokens, API keys, PII, credit card numbers.
- Use structured logging for security events to enable automated alerting.
- Implement rate limiting and monitor for anomalous patterns.

### Patterns

```python
import structlog

security_logger = structlog.get_logger("security")

def log_auth_event(
    event: str,
    *,
    user_id: str | None = None,
    ip_address: str,
    success: bool,
    reason: str = "",
) -> None:
    """Log a security-relevant authentication event."""
    security_logger.info(
        event,
        user_id=user_id,
        ip_address=ip_address,
        success=success,
        reason=reason,
    )

# Usage
log_auth_event("login_attempt", user_id="u123", ip_address=request.client.host, success=False, reason="invalid_password")
```

---

## 11. API Security

### Rules

- Always use HTTPS in production; reject plain HTTP.
- Implement **rate limiting** to prevent abuse and DDoS.
- Validate `Content-Type` headers on all incoming requests.
- Return generic error messages to clients; log detailed errors server-side.
- Use API versioning to manage breaking changes safely.
- Implement request size limits to prevent memory exhaustion.

### Rate Limiting

```python
# FastAPI with slowapi
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)

@app.get("/api/v1/data")
@limiter.limit("100/minute")
async def get_data(request: Request) -> DataResponse:
    ...
```

### CORS Configuration

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://yourdomain.com"],   # NEVER use ["*"] in production
    allow_methods=["GET", "POST"],
    allow_headers=["Authorization", "Content-Type"],
    allow_credentials=True,
    max_age=3600,
)
```

---

## 12. Serialization & Deserialization

### Rules

- **Never** use `pickle` to deserialize untrusted data — it allows arbitrary code execution.
- **Never** use `yaml.load()` — always use `yaml.safe_load()`.
- Prefer JSON for data interchange; use Pydantic for deserialization with validation.
- Avoid `eval()`, `exec()`, and `compile()` on any data derived from user input.

```python
import yaml

# DANGEROUS — allows arbitrary code execution
data = yaml.load(user_input)          # NEVER
data = pickle.loads(user_bytes)        # NEVER
result = eval(user_expression)         # NEVER

# SAFE
data = yaml.safe_load(user_input)      # OK
data = json.loads(user_input)          # OK (with Pydantic validation after)
model = MyModel.model_validate_json(user_input)  # BEST
```

---

## 13. Session & Cookie Security

### Rules

- Set `HttpOnly` flag on session cookies (prevents JavaScript access).
- Set `Secure` flag on cookies (transmitted only over HTTPS).
- Set `SameSite=Lax` or `SameSite=Strict` (prevents CSRF).
- Use short session expiration times and implement idle timeout.
- Regenerate session IDs after authentication state changes (login, privilege escalation).

```python
from starlette.responses import Response

def set_secure_cookie(response: Response, name: str, value: str) -> None:
    response.set_cookie(
        key=name,
        value=value,
        httponly=True,
        secure=True,
        samesite="lax",
        max_age=3600,  # 1 hour
        path="/",
    )
```

---

## 14. Error Handling for Security

### Rules

- Return generic error messages to clients; never expose stack traces, file paths, or internal details in production.
- Log the full exception server-side for debugging.
- Use custom exception handlers to transform internal errors into safe API responses.

```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse

app = FastAPI()

@app.exception_handler(Exception)
async def global_exception_handler(request: Request, exc: Exception) -> JSONResponse:
    # Log full details server-side
    logger.exception("Unhandled exception", exc_info=exc, path=request.url.path)
    # Return safe, generic message to client
    return JSONResponse(
        status_code=500,
        content={"detail": "An internal error occurred. Please try again later."},
    )
```

---

## 15. Security Checklist for Code Review

Use this checklist when reviewing code for security:

- [ ] **Input validation**: All external inputs validated and sanitized
- [ ] **Authentication**: Auth checks present on all protected endpoints
- [ ] **Authorization**: Role/permission checks enforced server-side
- [ ] **No hardcoded secrets**: No API keys, passwords, or tokens in code
- [ ] **Parameterized queries**: No SQL string formatting or concatenation
- [ ] **No dangerous functions**: No `eval()`, `exec()`, `pickle.loads()` on user data
- [ ] **Error handling**: No sensitive data leaked in error responses
- [ ] **Logging**: Security events logged; no sensitive data in logs
- [ ] **Dependencies**: No known vulnerabilities in dependencies
- [ ] **Crypto**: Using `secrets` for randomness; no weak algorithms
- [ ] **HTTPS**: All external communication over TLS
- [ ] **Headers**: Security headers set (CSP, HSTS, X-Content-Type-Options)
- [ ] **CORS**: Origins restricted to known domains
- [ ] **Rate limiting**: Abuse prevention on public endpoints
- [ ] **File handling**: Path traversal prevented; upload size/type limited

---

## OWASP Top 10 Quick Reference

| # | Risk | Key Mitigation |
|---|------|---------------|
| A01 | Broken Access Control | Server-side auth checks, least privilege, CORS |
| A02 | Cryptographic Failures | TLS everywhere, strong hashing, no hardcoded keys |
| A03 | Injection | Parameterized queries, input validation, no eval/exec |
| A04 | Insecure Design | Threat modeling, security requirements, secure defaults |
| A05 | Security Misconfiguration | Minimal permissions, security headers, no defaults |
| A06 | Vulnerable Components | Dependency audit, Dependabot, regular updates |
| A07 | Authentication Failures | MFA, strong passwords, account lockout, rate limiting |
| A08 | Data Integrity Failures | Signed updates, CI/CD security, input validation |
| A09 | Logging Failures | Security event logging, monitoring, alerting |
| A10 | SSRF | Validate URLs, allowlist destinations, network segmentation |

---

## Notes for LLM Agent Behavior

- **Always** generate input validation (Pydantic models or manual checks) for any function that accepts external input.
- **Never** suggest `eval()`, `exec()`, `pickle.loads()`, or `yaml.load()` in generated code.
- **Always** use parameterized queries — never string formatting for SQL.
- **Always** use `secrets` instead of `random` for security-sensitive values.
- When generating API endpoints, include authentication, authorization, rate limiting, and input validation.
- When generating error handlers, ensure no sensitive information is leaked to clients.
- When suggesting dependencies, check for known vulnerabilities and prefer well-maintained packages.
- Flag any hardcoded credentials, keys, or tokens found in code and recommend migration to environment variables or a secrets manager.
- When generating logging code, ensure no passwords, tokens, or PII are included in log messages.
- Default to the most secure option when multiple approaches exist.
