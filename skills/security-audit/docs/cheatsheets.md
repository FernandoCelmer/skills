# Security Audit Cheatsheets

## Quick Grep Patterns for Common Vulnerabilities

### Secrets & Credentials

```bash
# AWS keys
grep -rn "AKIA[0-9A-Z]\{16\}" .

# Generic secrets
grep -rn "(secret|password|token|api_key|apikey|auth)\s*=\s*['\"][^'\"]\{8,\}" .

# Private keys
grep -rn "BEGIN.*PRIVATE KEY" .

# Connection strings
grep -rn "(mongodb|mysql|postgres|redis|amqp)://[^\s'\"]*" .

# Hardcoded IPs (internal)
grep -rn "192\.168\.\|10\.\|172\.1[6-9]\.\|172\.2[0-9]\.\|172\.3[01]\." .
```

### SQL Injection

```bash
# Python - f-strings in queries
grep -rn "execute(f\"\|execute(f'\|\.format(" . --include="*.py"

# Python - string concatenation in queries
grep -rn "execute(.*+.*)" . --include="*.py"

# Raw SQL usage
grep -rn "\.raw(\|\.extra(\|text(" . --include="*.py"

# Node.js
grep -rn "query(\`\|query(\".*\$\{" . --include="*.js" --include="*.ts"
```

### Command Injection

```bash
# Python
grep -rn "os\.system\|os\.popen\|subprocess.*shell=True\|eval(\|exec(" . --include="*.py"

# Node.js
grep -rn "child_process\|exec(\|execSync(" . --include="*.js" --include="*.ts"

# Ruby
grep -rn "system(\|exec(\|\`.*\#\{" . --include="*.rb"
```

### XSS

```bash
# React
grep -rn "dangerouslySetInnerHTML" . --include="*.jsx" --include="*.tsx"

# Vue
grep -rn "v-html" . --include="*.vue"

# Angular
grep -rn "bypassSecurityTrust\|innerHTML" . --include="*.ts" --include="*.html"

# Generic
grep -rn "\.innerHTML\s*=" . --include="*.js" --include="*.ts"
```

### Insecure Deserialization

```bash
# Python
grep -rn "pickle\.loads\|pickle\.load\|yaml\.load(\|yaml\.unsafe_load\|marshal\.loads" . --include="*.py"

# Java
grep -rn "ObjectInputStream\|readObject()\|XMLDecoder" . --include="*.java"

# Node.js
grep -rn "serialize\|node-serialize\|unserialize" . --include="*.js"

# PHP
grep -rn "unserialize(" . --include="*.php"
```

### Path Traversal

```bash
# File operations with user input
grep -rn "open(\|read_file\|send_file\|send_from_directory" . --include="*.py"

# Node.js
grep -rn "fs\.readFile\|fs\.createReadStream\|path\.join.*req\." . --include="*.js" --include="*.ts"

# Check for ../ sanitization
grep -rn "\.\.\/" . --include="*.py" --include="*.js" --include="*.ts"
```

### SSRF

```bash
# Python
grep -rn "requests\.get\|requests\.post\|urllib\.request\|httpx\.\|aiohttp" . --include="*.py"

# Node.js
grep -rn "fetch(\|axios\.\|got(\|request(" . --include="*.js" --include="*.ts"

# Check if URL comes from user input
grep -rn "request\.\(json\|args\|form\|data\).*url\|request\..*\(get\|post\)" . --include="*.py"
```

### Authentication Issues

```bash
# JWT without verification
grep -rn "jwt\.decode.*verify=False\|algorithms=\[\"none\"\]\|verify_signature.*False" . --include="*.py"

# Weak comparison
grep -rn "==.*password\|password.*==" . --include="*.py"

# Missing auth decorators
grep -rn "@app\.\(get\|post\|put\|delete\|patch\)" . --include="*.py" | grep -v "login\|signup\|public\|health"
```

### CORS Issues

```bash
grep -rn "Access-Control-Allow-Origin.*\*\|origins.*\*\|allow_origins.*\*" .
grep -rn "CORS(\|cors(\|CORSMiddleware" .
```

### Cryptography

```bash
# Weak hashing
grep -rn "md5(\|sha1(\|hashlib\.md5\|hashlib\.sha1" . --include="*.py"
grep -rn "crypto\.createHash.*md5\|crypto\.createHash.*sha1" . --include="*.js"

# Weak random
grep -rn "random\.random\|random\.randint\|Math\.random" . --include="*.py" --include="*.js"

# ECB mode
grep -rn "AES.*ECB\|MODE_ECB\|mode.*ecb" .
```

---

## Framework-Specific Checks

### Django

```bash
# Debug mode
grep -rn "DEBUG\s*=\s*True" . --include="*.py"

# CSRF disabled
grep -rn "@csrf_exempt\|CSRF_COOKIE_SECURE.*False" . --include="*.py"

# SQL injection via extra/raw
grep -rn "\.extra(\|\.raw(" . --include="*.py"

# Secret key exposed
grep -rn "SECRET_KEY\s*=" . --include="*.py" --include="*.env"
```

### FastAPI / Flask

```bash
# Missing input validation
grep -rn "request\.json\|request\.args\|request\.form" . --include="*.py"

# No rate limiting
grep -rn "@app\.\(get\|post\|put\|delete\)" . --include="*.py" | wc -l
grep -rn "slowapi\|flask-limiter\|ratelimit" . --include="*.py" | wc -l

# CORS wildcard
grep -rn "CORSMiddleware\|CORS(" . --include="*.py"
```

### Express / Node.js

```bash
# No helmet (security headers)
grep -rn "helmet\|x-frame-options\|x-content-type" . --include="*.js" --include="*.ts"

# SQL injection
grep -rn "query(\`\|query(\"" . --include="*.js" --include="*.ts" | grep -v "parameterized\|placeholder"

# File upload no validation
grep -rn "multer\|formidable\|busboy" . --include="*.js" --include="*.ts"
```

### React / Frontend

```bash
# Sensitive data in client
grep -rn "localStorage\.setItem.*token\|localStorage.*password\|localStorage.*secret" . --include="*.js" --include="*.ts" --include="*.jsx" --include="*.tsx"

# Eval usage
grep -rn "eval(\|Function(\|setTimeout(\"" . --include="*.js" --include="*.ts"

# Console logs with sensitive data
grep -rn "console\.log.*token\|console\.log.*password\|console\.log.*secret" . --include="*.js" --include="*.ts"
```

---

## Docker Security Checklist

```bash
# Running as root?
grep -L "^USER" */Dockerfile

# Using latest tag?
grep "FROM.*:latest" */Dockerfile

# Secrets in build?
grep -n "ARG.*SECRET\|ARG.*PASSWORD\|ARG.*TOKEN\|ENV.*SECRET\|ENV.*PASSWORD" */Dockerfile

# Exposed ports?
grep -n "EXPOSE" */Dockerfile

# Health check?
grep -L "HEALTHCHECK" */Dockerfile

# .dockerignore exists?
ls .dockerignore 2>/dev/null || echo "MISSING .dockerignore"

# Sensitive files not ignored?
grep -v ".env\|.git\|node_modules\|__pycache__\|*.key\|*.pem" .dockerignore
```

---

## CI/CD Security Checklist

```bash
# Secrets in plain text
grep -rn "password:\|secret:\|token:\|api_key:" .github/workflows/ | grep -v "\$\{\{.*secrets\."

# pull_request_target danger
grep -rn "pull_request_target" .github/workflows/

# Pinned actions?
grep -rn "uses:" .github/workflows/ | grep -v "@[a-f0-9]\{40\}\|@v[0-9]"

# Write permissions
grep -rn "permissions:.*write\|contents: write" .github/workflows/

# Self-hosted runners
grep -rn "self-hosted" .github/workflows/
```

---

## HTTP Security Headers Checklist

| Header | Expected Value |
|--------|---------------|
| Strict-Transport-Security | `max-age=31536000; includeSubDomains` |
| Content-Security-Policy | Restrict sources, no `unsafe-inline` |
| X-Content-Type-Options | `nosniff` |
| X-Frame-Options | `DENY` or `SAMEORIGIN` |
| X-XSS-Protection | `0` (deprecated, rely on CSP) |
| Referrer-Policy | `strict-origin-when-cross-origin` |
| Permissions-Policy | Restrict camera, microphone, geolocation |
| Cache-Control | `no-store` for sensitive responses |
