# OWASP Top 10 (2021) — Detailed Reference

## A01:2021 — Broken Access Control

### Description
Access control enforces policy such that users cannot act outside of their intended permissions.

### Common Vulnerabilities
- Violation of least privilege or deny by default
- Bypassing access control by modifying URL, internal state, HTML page, or API requests
- Permitting viewing or editing someone else's account (IDOR)
- Accessing API with missing access controls for POST, PUT, DELETE
- Elevation of privilege (acting as user without login, or as admin when logged in as user)
- Metadata manipulation (replaying/tampering JWT, cookie, hidden field)
- CORS misconfiguration allowing API access from unauthorized origins
- Force browsing to authenticated/privileged pages

### Detection Patterns

```python
# IDOR — No ownership verification
@app.get("/api/orders/{order_id}")
def get_order(order_id: int, db: Session):
    return db.query(Order).get(order_id)  # Missing: WHERE user_id = current_user.id

# Missing auth decorator
@app.delete("/api/admin/users/{user_id}")
def delete_user(user_id: int):  # No @require_admin or auth check
    db.delete(user_id)

# Mass assignment
@app.patch("/api/profile")
def update_profile(data: dict):
    user.update(**data)  # Accepts role, is_admin, credits...
```

### Prevention
- Deny by default (except public resources)
- Implement access control once, reuse throughout application
- Model access controls enforce record ownership
- Disable web server directory listing
- Log access control failures, alert admins
- Rate limit API and controller access
- Stateless JWT tokens should be short-lived

### References
- https://owasp.org/Top10/A01_2021-Broken_Access_Control/
- CWE-200, CWE-284, CWE-285, CWE-352, CWE-639

---

## A02:2021 — Cryptographic Failures

### Description
Failures related to cryptography which often lead to sensitive data exposure.

### Common Vulnerabilities
- Transmitting data in clear text (HTTP, SMTP, FTP)
- Using old/weak cryptographic algorithms (MD5, SHA1, DES, RC4)
- Using default crypto keys, weak keys, or reusing keys
- Not enforcing encryption (missing HSTS, TLS directives)
- Not validating server certificate chain
- Using deprecated hash functions for passwords
- Missing proper key management/rotation

### Detection Patterns

```python
# Weak password hashing
import hashlib
password_hash = hashlib.md5(password.encode()).hexdigest()  # WEAK
password_hash = hashlib.sha256(password.encode()).hexdigest()  # NO SALT

# Should be:
from passlib.hash import bcrypt
password_hash = bcrypt.hash(password)

# Hardcoded encryption key
ENCRYPTION_KEY = "my-secret-key-123"  # IN SOURCE CODE
AES_KEY = b'\x00' * 16  # Null key

# Weak randomness
import random
token = ''.join(random.choices(string.ascii_letters, k=32))  # PREDICTABLE

# Should be:
import secrets
token = secrets.token_urlsafe(32)
```

### Prevention
- Classify data (PII, financial, health) and apply controls per classification
- Don't store sensitive data unnecessarily
- Encrypt all data in transit (TLS 1.2+) and at rest
- Use strong adaptive salted hashing for passwords (bcrypt, Argon2, scrypt)
- Use authenticated encryption (AES-GCM, ChaCha20-Poly1305)
- Generate keys with cryptographically secure PRNG

### References
- https://owasp.org/Top10/A02_2021-Cryptographic_Failures/
- CWE-259, CWE-327, CWE-328, CWE-330, CWE-916

---

## A03:2021 — Injection

### Description
User-supplied data is not validated, filtered, or sanitized by the application.

### Types
- SQL, NoSQL, OS Command, ORM, LDAP, EL/OGNL, Template (SSTI)

### Detection Patterns

```python
# SQL Injection
query = f"SELECT * FROM users WHERE email = '{email}'"
cursor.execute(query)

# Command Injection
os.system(f"nslookup {domain}")
subprocess.Popen(f"ffmpeg -i {filename} output.mp4", shell=True)

# Template Injection (SSTI)
from jinja2 import Template
Template(user_input).render()  # User controls template

# NoSQL Injection (MongoDB)
db.users.find({"username": request.json["username"]})
# Attacker sends: {"username": {"$gt": ""}}

# XSS
return f"<h1>Hello {username}</h1>"  # Reflected XSS
element.innerHTML = userInput;  # DOM XSS
```

### Prevention
- Use parameterized queries / prepared statements
- Use ORMs (but beware `.raw()`, `.extra()`)
- Server-side input validation (allowlist)
- Escape special characters for specific interpreter
- Use LIMIT and other SQL controls to prevent mass disclosure

### References
- https://owasp.org/Top10/A03_2021-Injection/
- CWE-20, CWE-74, CWE-75, CWE-77, CWE-78, CWE-79, CWE-89, CWE-94, CWE-643

---

## A04:2021 — Insecure Design

### Description
Risks related to design and architectural flaws, calling for more use of threat modeling and secure design patterns.

### Common Issues
- Missing rate limiting on expensive operations
- No anti-automation for credential recovery
- Missing business logic validation
- Trusting client-side validation only
- No abuse case modeling

### Detection Patterns

```python
# No rate limit on sensitive operation
@app.post("/api/auth/login")
def login(credentials):  # Can be brute-forced infinitely
    ...

# Trusting client-side price
@app.post("/api/checkout")
def checkout(data):
    total = data["price"]  # Price from client, not recalculated

# No idempotency
@app.post("/api/transfer")
def transfer(data):  # Can be replayed
    debit(data["from"], data["amount"])
    credit(data["to"], data["amount"])
```

### Prevention
- Establish secure development lifecycle with AppSec professionals
- Use threat modeling for authentication, access control, business logic
- Write unit and integration tests to validate all critical flows
- Design for the adversary — assume any client input is hostile

### References
- https://owasp.org/Top10/A04_2021-Insecure_Design/
- CWE-209, CWE-256, CWE-501, CWE-522

---

## A05:2021 — Security Misconfiguration

### Description
Missing security hardening, unnecessary features enabled, default accounts, overly informative error messages.

### Detection Patterns

```yaml
# Docker running as root (no USER directive)
FROM python:3.11
COPY . /app
CMD ["python", "app.py"]  # Runs as root!

# Debug mode in production
DEBUG = True  # Django/Flask
app.run(debug=True)

# Permissive CORS
CORS(app, origins="*", supports_credentials=True)

# Default credentials
DB_USER = "admin"
DB_PASS = "admin"

# Unnecessary features
INSTALLED_APPS = [..., 'debug_toolbar', ...]  # In production
```

```yaml
# CI/CD - secrets exposed
env:
  AWS_SECRET_KEY: "AKIAIOSFODNN7EXAMPLE"  # Hardcoded in workflow
```

### Prevention
- Minimal platform without unnecessary features/components
- Review and update configurations as part of patch management
- Segmented application architecture
- Send security directives to clients (HSTS, CSP, X-Frame-Options)
- Automated process to verify configuration effectiveness

### References
- https://owasp.org/Top10/A05_2021-Security_Misconfiguration/
- CWE-2, CWE-11, CWE-13, CWE-15, CWE-16, CWE-388

---

## A06:2021 — Vulnerable and Outdated Components

### Description
Using components with known vulnerabilities (libraries, frameworks, OS).

### Detection

```bash
# Python
pip audit
safety check
poetry show --outdated

# Node
npm audit
yarn audit
npx retire

# Go
govulncheck ./...

# General
gh api repos/OWNER/REPO/dependabot/alerts
```

### Prevention
- Remove unused dependencies
- Continuously inventory component versions
- Monitor CVE databases (NVD, GitHub Advisory)
- Only obtain components from official sources over secure links
- Pin versions in lock files

### References
- https://owasp.org/Top10/A06_2021-Vulnerable_and_Outdated_Components/
- CWE-1035, CWE-1104

---

## A07:2021 — Identification and Authentication Failures

### Description
Confirmation of identity, authentication, and session management weaknesses.

### Detection Patterns

```python
# Weak password policy
if len(password) >= 4:  # Too short!
    create_user(password)

# No brute force protection
@app.post("/login")
def login(email, password):  # No rate limit, no lockout
    user = db.query(User).filter_by(email=email).first()
    if user and check_password(password, user.password_hash):
        return create_token(user)
    return {"error": "Invalid credentials"}

# Timing attack - different response times
if not user:
    return error  # Fast path
if not check_password(password, user.hash):
    return error  # Slow path (bcrypt comparison)

# Session not invalidated on logout
@app.post("/logout")
def logout():
    return {"message": "logged out"}  # Token still valid!
```

### Prevention
- Multi-factor authentication
- Do not deploy with default credentials
- Implement weak-password checks (top 10000 passwords)
- Align password policies with NIST 800-63b
- Limit failed login attempts (lockout + exponential backoff)
- Use constant-time comparison for auth checks
- Generate high-entropy session IDs server-side

### References
- https://owasp.org/Top10/A07_2021-Identification_and_Authentication_Failures/
- CWE-255, CWE-259, CWE-287, CWE-288, CWE-307, CWE-384

---

## A08:2021 — Software and Data Integrity Failures

### Description
Assumptions about software updates, critical data, CI/CD pipelines without verifying integrity.

### Detection Patterns

```python
# Insecure deserialization
import pickle
data = pickle.loads(user_input)  # RCE!

import yaml
data = yaml.load(user_input)  # Should be yaml.safe_load()

# CI/CD pipeline injection
# GitHub Actions with pull_request_target + checkout PR code
on: pull_request_target
steps:
  - uses: actions/checkout@v4
    with:
      ref: ${{ github.event.pull_request.head.sha }}  # DANGEROUS
  - run: make test  # Runs attacker's Makefile
```

### Prevention
- Use digital signatures to verify software/data from expected source
- Use `pip --require-hashes`, npm integrity checks, lock files
- Do not use `pickle`/`marshal` with untrusted input (use JSON)
- Use `yaml.safe_load()` instead of `yaml.load()`
- Ensure CI/CD pipelines have proper access control and segregation

### References
- https://owasp.org/Top10/A08_2021-Software_and_Data_Integrity_Failures/
- CWE-345, CWE-353, CWE-426, CWE-494, CWE-502, CWE-565

---

## A09:2021 — Security Logging and Monitoring Failures

### Description
Without logging and monitoring, breaches cannot be detected.

### Detection Patterns

```python
# Sensitive data in logs
logger.info(f"User login: {email}, password: {password}")
logger.debug(f"Token: {jwt_token}")
logger.error(f"DB query failed: {full_sql_with_params}")

# No audit logging on sensitive operations
@app.delete("/api/users/{user_id}")
def delete_user(user_id):
    db.delete(user_id)
    return {"ok": True}  # No log of WHO deleted WHAT

# No alerting
# Application has no mechanism to detect active attacks
```

### Prevention
- Log all login, access control, input validation failures
- Ensure logs have enough context for forensics
- Ensure log data is encoded correctly to prevent injection
- Establish effective monitoring and alerting
- Don't log sensitive data (passwords, tokens, PII)

### References
- https://owasp.org/Top10/A09_2021-Security_Logging_and_Monitoring_Failures/
- CWE-117, CWE-223, CWE-532, CWE-778

---

## A10:2021 — Server-Side Request Forgery (SSRF)

### Description
SSRF occurs when a web application fetches a remote resource without validating the user-supplied URL.

### Detection Patterns

```python
# SSRF - fetching user-supplied URL
import requests

@app.post("/api/fetch-url")
def fetch_url(data):
    url = data["url"]  # User controls destination
    response = requests.get(url)  # Can access internal services!
    return response.text

# Can be exploited to access:
# http://169.254.169.254/latest/meta-data/ (AWS metadata)
# http://localhost:8080/admin (internal services)
# file:///etc/passwd (local files)

# Partial SSRF via redirects
# Server validates domain but follows redirect to internal IP
```

### Prevention
- Sanitize and validate all client-supplied input data
- Enforce URL schema, port, and destination with allowlist
- Do not send raw responses to clients
- Disable HTTP redirections
- Use network segmentation (firewalls deny by default)
- Block access to metadata endpoints (169.254.169.254)

### References
- https://owasp.org/Top10/A10_2021-Server-Side_Request_Forgery_%28SSRF%29/
- CWE-918
