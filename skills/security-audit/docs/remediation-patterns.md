# Remediation Patterns — Secure Code Examples

Quick-reference fixes for common vulnerabilities.

---

## SQL Injection → Parameterized Queries

### Python (SQLAlchemy)
```python
# VULNERABLE
db.execute(f"SELECT * FROM users WHERE email = '{email}'")

# SECURE
db.execute(text("SELECT * FROM users WHERE email = :email"), {"email": email})

# ORM (preferred)
db.query(User).filter(User.email == email).first()
```

### Python (psycopg2 / mysql-connector)
```python
# VULNERABLE
cursor.execute(f"SELECT * FROM users WHERE id = {user_id}")

# SECURE
cursor.execute("SELECT * FROM users WHERE id = %s", (user_id,))
```

### Node.js (pg)
```javascript
// VULNERABLE
client.query(`SELECT * FROM users WHERE id = ${userId}`)

// SECURE
client.query('SELECT * FROM users WHERE id = $1', [userId])
```

---

## Command Injection → Avoid shell=True

### Python
```python
# VULNERABLE
os.system(f"convert {filename} output.png")
subprocess.run(f"ping {host}", shell=True)

# SECURE
subprocess.run(["convert", filename, "output.png"], shell=False)
subprocess.run(["ping", "-c", "4", host], shell=False)

# If shell features needed, use shlex
import shlex
subprocess.run(shlex.split(f"command --arg={shlex.quote(user_input)}"))
```

---

## XSS → Output Encoding

### React
```jsx
// VULNERABLE
<div dangerouslySetInnerHTML={{__html: userContent}} />

// SECURE — React auto-escapes by default
<div>{userContent}</div>

// If HTML needed, sanitize first
import DOMPurify from 'dompurify';
<div dangerouslySetInnerHTML={{__html: DOMPurify.sanitize(userContent)}} />
```

### Python (Jinja2)
```python
# VULNERABLE
Template(user_input).render()
return f"<h1>{user_input}</h1>"

# SECURE — auto-escaping enabled
from markupsafe import escape
return f"<h1>{escape(user_input)}</h1>"

# Jinja2 with autoescape (default in Flask)
env = Environment(autoescape=True)
```

---

## Path Traversal → Validate & Restrict

```python
# VULNERABLE
filepath = os.path.join(UPLOAD_DIR, user_filename)
return send_file(filepath)

# SECURE
from pathlib import Path

filepath = Path(UPLOAD_DIR) / user_filename
filepath = filepath.resolve()

# Ensure path is within allowed directory
if not str(filepath).startswith(str(Path(UPLOAD_DIR).resolve())):
    raise ValueError("Invalid path")

return send_file(filepath)
```

---

## SSRF → URL Allowlist

```python
# VULNERABLE
response = requests.get(user_url)

# SECURE
from urllib.parse import urlparse
import ipaddress

ALLOWED_HOSTS = {"api.example.com", "cdn.example.com"}
BLOCKED_RANGES = [
    ipaddress.ip_network("10.0.0.0/8"),
    ipaddress.ip_network("172.16.0.0/12"),
    ipaddress.ip_network("192.168.0.0/16"),
    ipaddress.ip_network("169.254.0.0/16"),  # AWS metadata
    ipaddress.ip_network("127.0.0.0/8"),
]

def safe_fetch(url: str) -> requests.Response:
    parsed = urlparse(url)

    if parsed.scheme not in ("http", "https"):
        raise ValueError("Invalid scheme")

    if parsed.hostname not in ALLOWED_HOSTS:
        # Resolve and check IP
        ip = ipaddress.ip_address(socket.gethostbyname(parsed.hostname))
        if any(ip in network for network in BLOCKED_RANGES):
            raise ValueError("Blocked IP range")

    return requests.get(url, allow_redirects=False, timeout=10)
```

---

## Authentication — Secure Patterns

### Password Hashing
```python
# VULNERABLE
import hashlib
hash = hashlib.sha256(password.encode()).hexdigest()

# SECURE
from passlib.hash import argon2
hash = argon2.hash(password)
verified = argon2.verify(password, hash)

# Or bcrypt
from passlib.hash import bcrypt
hash = bcrypt.hash(password)
```

### JWT
```python
# VULNERABLE
token = jwt.encode(payload, "weak-secret", algorithm="HS256")
decoded = jwt.decode(token, options={"verify_signature": False})

# SECURE
import secrets

SECRET = secrets.token_urlsafe(64)  # Strong secret, from env var

token = jwt.encode(
    {**payload, "exp": datetime.utcnow() + timedelta(minutes=15)},
    SECRET,
    algorithm="HS256"
)

decoded = jwt.decode(
    token,
    SECRET,
    algorithms=["HS256"],  # Explicit algorithm list
    options={"require": ["exp", "sub"]}
)
```

### Rate Limiting
```python
# FastAPI with slowapi
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)

@app.post("/api/auth/login")
@limiter.limit("5/minute")
def login(request: Request, credentials: LoginSchema):
    ...
```

### Constant-Time Comparison
```python
# VULNERABLE — timing attack
if user_token == stored_token:
    ...

# SECURE
import hmac
if hmac.compare_digest(user_token, stored_token):
    ...
```

---

## Authorization — Ownership Checks

```python
# VULNERABLE — IDOR
@app.get("/api/orders/{order_id}")
def get_order(order_id: int, db: Session):
    return db.query(Order).get(order_id)

# SECURE
@app.get("/api/orders/{order_id}")
def get_order(order_id: int, db: Session, current_user: User = Depends(get_current_user)):
    order = db.query(Order).filter(
        Order.id == order_id,
        Order.user_id == current_user.id  # Ownership check
    ).first()
    if not order:
        raise HTTPException(404)
    return order
```

---

## File Upload — Validation

```python
# SECURE file upload
ALLOWED_EXTENSIONS = {".jpg", ".jpeg", ".png", ".gif", ".pdf"}
MAX_SIZE = 10 * 1024 * 1024  # 10MB

async def upload_file(file: UploadFile):
    # Check extension
    ext = Path(file.filename).suffix.lower()
    if ext not in ALLOWED_EXTENSIONS:
        raise HTTPException(400, "Invalid file type")

    # Check size
    content = await file.read()
    if len(content) > MAX_SIZE:
        raise HTTPException(400, "File too large")

    # Check magic bytes (not just extension)
    import magic
    mime = magic.from_buffer(content, mime=True)
    if mime not in {"image/jpeg", "image/png", "image/gif", "application/pdf"}:
        raise HTTPException(400, "Invalid content type")

    # Generate safe filename (never use user's filename)
    safe_name = f"{uuid4()}{ext}"
    save_path = Path(UPLOAD_DIR) / safe_name

    # Write file
    save_path.write_bytes(content)
    return {"filename": safe_name}
```

---

## CORS — Strict Configuration

```python
# VULNERABLE
app.add_middleware(CORSMiddleware, allow_origins=["*"], allow_credentials=True)

# SECURE
ALLOWED_ORIGINS = [
    "https://app.example.com",
    "https://www.example.com",
]

app.add_middleware(
    CORSMiddleware,
    allow_origins=ALLOWED_ORIGINS,
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "DELETE"],
    allow_headers=["Authorization", "Content-Type"],
    max_age=3600,
)
```

---

## Security Headers

```python
# FastAPI middleware
@app.middleware("http")
async def security_headers(request: Request, call_next):
    response = await call_next(request)
    response.headers["Strict-Transport-Security"] = "max-age=31536000; includeSubDomains"
    response.headers["Content-Security-Policy"] = "default-src 'self'; script-src 'self'"
    response.headers["X-Content-Type-Options"] = "nosniff"
    response.headers["X-Frame-Options"] = "DENY"
    response.headers["Referrer-Policy"] = "strict-origin-when-cross-origin"
    response.headers["Permissions-Policy"] = "camera=(), microphone=(), geolocation=()"
    return response
```

---

## Logging — Safe Patterns

```python
# VULNERABLE — sensitive data in logs
logger.info(f"Login attempt: email={email}, password={password}")
logger.debug(f"JWT: {token}")
logger.error(f"Query failed: {sql_with_params}")

# SECURE
logger.info(f"Login attempt: email={email}")  # No password
logger.debug("JWT issued for user_id=%s", user_id)  # No token value
logger.error("Query failed: %s", query_name)  # No params/data

# Structured logging with redaction
import structlog
logger = structlog.get_logger()
logger.info("login_attempt", email=email, ip=request.client.host)
```
