# Advanced Web Vulnerabilities — Code-Level Detection

Topics beyond OWASP Top 10 that are detectable through source code analysis.

---

## Race Conditions (TOCTOU)

### What to look for
- Check-then-act without locking (balance check → debit)
- File existence check → file operation
- Database read → conditional write without row-level lock

### Detection patterns

```python
# VULNERABLE — race on balance
def transfer(from_id, to_id, amount):
    sender = db.query(User).get(from_id)
    if sender.balance >= amount:  # CHECK
        sender.balance -= amount   # ACT — another thread may have already debited
        db.commit()

# SECURE — SELECT FOR UPDATE
def transfer(from_id, to_id, amount):
    sender = db.query(User).with_for_update().get(from_id)
    if sender.balance >= amount:
        sender.balance -= amount
        db.commit()
```

```bash
# Grep for race-prone patterns
grep -rn "if.*balance\|if.*count\|if.*quantity" . --include="*.py" | grep -v "lock\|atomic\|for_update"
```

---

## ReDoS (Regular Expression Denial of Service)

### What to look for
- Nested quantifiers: `(a+)+`, `(a|a)*`
- Overlapping alternations: `(a|ab)*`
- User input validated by complex regex

### Detection patterns

```python
# VULNERABLE — catastrophic backtracking
import re
pattern = re.compile(r"^(a+)+$")  # ReDoS
pattern = re.compile(r"^([a-zA-Z0-9]+)*@")  # ReDoS on email

# SECURE — use atomic groups or limit input length
if len(user_input) > 1000:
    raise ValueError("Input too long")
pattern = re.compile(r"^[a-zA-Z0-9]+@")  # No nested quantifiers
```

```bash
# Grep for dangerous regex patterns
grep -rn "re\.compile\|re\.match\|re\.search\|re\.sub" . --include="*.py"
grep -rn "new RegExp\|\.match(\|\.test(" . --include="*.js" --include="*.ts"
# Then manually check for nested quantifiers
```

---

## XXE (XML External Entity)

### What to look for
- XML parsing without disabling external entities
- SOAP/XML-RPC endpoints
- File uploads accepting XML/SVG/DOCX

### Detection patterns

```python
# VULNERABLE
from xml.etree.ElementTree import parse
tree = parse(user_file)  # XXE possible

import lxml.etree
parser = lxml.etree.XMLParser()  # resolve_entities=True by default
tree = lxml.etree.parse(user_file, parser)

# SECURE
parser = lxml.etree.XMLParser(
    resolve_entities=False,
    no_network=True,
    dtd_validation=False,
)
tree = lxml.etree.parse(user_file, parser)

# Or use defusedxml
from defusedxml.ElementTree import parse
tree = parse(user_file)  # Safe by default
```

```bash
grep -rn "xml\.etree\|lxml\.etree\|xml\.dom\|xml\.sax\|XMLParser" . --include="*.py"
grep -rn "DocumentBuilder\|SAXParser\|XMLReader" . --include="*.java"
```

---

## Open Redirect

### What to look for
- Redirect URL from user parameter without allowlist validation
- `next=`, `redirect=`, `return_to=`, `url=` parameters

### Detection patterns

```python
# VULNERABLE
@app.get("/login")
def login(next: str = "/"):
    ...
    return RedirectResponse(next)  # Attacker: /login?next=https://evil.com

# SECURE
ALLOWED_HOSTS = {"app.example.com", "www.example.com"}

def safe_redirect(url: str) -> str:
    parsed = urlparse(url)
    if parsed.netloc and parsed.netloc not in ALLOWED_HOSTS:
        return "/"
    if not url.startswith("/"):
        return "/"
    return url
```

```bash
grep -rn "redirect\|RedirectResponse\|res\.redirect\|location.*=.*request\|next=\|return_to" . --include="*.py" --include="*.js" --include="*.ts"
```

---

## Cookie Security

### What to look for
- Missing `Secure` flag (sent over HTTP)
- Missing `HttpOnly` flag (accessible via JS)
- Missing `SameSite` attribute (CSRF)
- Sensitive data in cookies without encryption

### Detection patterns

```python
# VULNERABLE
response.set_cookie("session_id", token)  # No flags!

# SECURE
response.set_cookie(
    "session_id",
    token,
    httponly=True,
    secure=True,
    samesite="Lax",
    max_age=3600,
    domain=".example.com",
)
```

```bash
grep -rn "set_cookie\|setCookie\|cookie(" . --include="*.py" --include="*.js" --include="*.ts" | grep -v "httponly\|secure\|samesite"
```

---

## WebSocket Security

### What to look for
- No authentication on WS connection
- No origin validation
- No message size limits
- Sensitive data without encryption

### Detection patterns

```python
# VULNERABLE — no auth on WS
@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    await websocket.accept()  # No token verification!
    ...

# SECURE
@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    token = websocket.query_params.get("token")
    if not verify_token(token):
        await websocket.close(code=4001)
        return
    await websocket.accept()
```

```bash
grep -rn "websocket\|WebSocket\|ws://" . --include="*.py" --include="*.js" --include="*.ts" | grep -v "wss://"
grep -rn "\.accept()\|onopen\|on_connect" . | grep -v "auth\|token\|verify"
```

---

## Dependency Confusion

### What to look for
- Internal package names in requirements that don't exist on public PyPI/npm
- Missing `--index-url` restriction
- No hash pinning

### Detection patterns

```bash
# Check if packages exist on public registry
# Python
cat requirements.txt | awk -F'[=<>!]' '{print $1}' | while read pkg; do
    pip index versions "$pkg" 2>/dev/null || echo "PRIVATE?: $pkg"
done

# Node — check for scoped packages without scope
grep -v "^@" package.json | grep -o '"[^"]*":' | grep -v "version\|name\|description"
```

---

## UUID Insecurities

### What to look for
- UUID v1 used for tokens/session IDs (contains MAC address + timestamp)
- Sequential UUIDs as access tokens
- UUID as sole authorization mechanism

### Detection patterns

```python
# VULNERABLE — UUID v1 leaks MAC + time
import uuid
session_id = str(uuid.uuid1())  # Predictable + info leak

# SECURE
session_id = str(uuid.uuid4())  # Random
# Or better:
import secrets
session_id = secrets.token_urlsafe(32)
```

```bash
grep -rn "uuid\.uuid1\|uuid1()\|uuidv1\|UUID.*version=1" . --include="*.py" --include="*.js"
```

---

## 2FA Bypass Patterns

### What to look for in code
- 2FA check skipped on certain endpoints
- 2FA code with no rate limit
- 2FA code valid too long (>5 min)
- Response manipulation (status code check client-side)
- Backup codes not rate-limited

### Detection patterns

```python
# VULNERABLE — 2FA check only on login, not on sensitive actions
@app.post("/api/transfer")
@require_auth  # Only checks JWT, not 2FA status
def transfer(data):
    ...

# SECURE — verify 2FA for sensitive ops
@app.post("/api/transfer")
@require_auth
@require_2fa  # Additional check
def transfer(data):
    ...
```

```bash
# Find endpoints that might need 2FA but don't have it
grep -rn "@require_auth\|@login_required\|Depends(get_current_user)" . --include="*.py" | grep -v "2fa\|mfa\|otp"
```

---

## HTTP Request Smuggling (Code-Level)

### What to look for
- Custom HTTP parsing logic
- Proxy/load balancer configs in repo
- `Transfer-Encoding` / `Content-Length` handling
- NGINX/Apache configs allowing ambiguous requests

### Detection patterns

```bash
# Check proxy configs
grep -rn "proxy_pass\|ProxyPass\|upstream" . --include="*.conf" --include="*.nginx"
grep -rn "transfer-encoding\|content-length\|chunked" . --include="*.py" --include="*.js"

# Check for HTTP/2 downgrade configs
grep -rn "h2c\|HTTP2\|http2_push" . --include="*.conf"
```

---

## Cache Poisoning / Deception

### What to look for
- Caching responses that contain user-specific data
- Cache key not including all relevant headers
- `Vary` header missing or incomplete

### Detection patterns

```python
# VULNERABLE — caching with user-specific content
@app.get("/profile")
@cache(ttl=300)  # Caches response per URL, not per user!
def profile(current_user):
    return {"name": current_user.name, "email": current_user.email}

# SECURE — cache key includes user
@app.get("/profile")
@cache(ttl=300, key_func=lambda req: f"{req.url}:{req.user.id}")
def profile(current_user):
    ...
```

```bash
grep -rn "@cache\|Cache-Control\|CDN\|Varnish\|CloudFront\|redis.*cache" . --include="*.py" --include="*.js" --include="*.conf"
grep -rn "Vary:" . --include="*.py" --include="*.conf"
```

---

## Timing Attacks

### What to look for
- String comparison for secrets/tokens using `==`
- Different code paths for valid/invalid users (enumeration)
- Database queries that reveal existence by timing

### Detection patterns

```python
# VULNERABLE
if provided_token == stored_token:  # Timing leak
    return True

if not User.query.filter_by(email=email).first():  # Fast path
    return error
if not check_password(password, user.hash):  # Slow path (bcrypt)
    return error

# SECURE
import hmac
if hmac.compare_digest(provided_token.encode(), stored_token.encode()):
    return True

# Always compute hash even if user doesn't exist
user = User.query.filter_by(email=email).first()
dummy_hash = "$2b$12$LJ3m4sMKlXNGdX7Ax1234OvY..."
check_password(password, user.hash if user else dummy_hash)
```

```bash
grep -rn "==.*token\|==.*secret\|==.*api_key\|password.*==" . --include="*.py" | grep -v "hmac\|compare_digest\|constant_time"
```

---

## Mass Assignment (CWE-915)

### What to look for
- ORM/model updated with raw request data
- No allowlist of updatable fields
- Admin fields (role, is_admin, credits) modifiable

### Detection patterns

```python
# VULNERABLE
user.update(**request.json)
User.query.filter_by(id=user_id).update(request.json)
serializer = UserSerializer(data=request.data)  # All fields accepted

# SECURE — explicit allowlist
ALLOWED_FIELDS = {"name", "email", "bio"}
data = {k: v for k, v in request.json.items() if k in ALLOWED_FIELDS}
user.update(**data)
```

```bash
grep -rn "update(\*\*\|\.update(request\|\.update(data\|\.update(body" . --include="*.py" --include="*.js"
grep -rn "Object\.assign.*req\.body\|{\.\.\.req\.body}" . --include="*.js" --include="*.ts"
```
