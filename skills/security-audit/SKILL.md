---
allowed-tools: Bash, Read, Glob, Grep, Agent, WebFetch, Write
description: "Security audit skill for repositories. Use when the user asks to 'audit security', 'pentest repo', 'find vulnerabilities', 'security review', 'check for exploits', 'OWASP check', or wants a comprehensive security analysis of a codebase. Covers OWASP Top 10, dependency vulnerabilities, secrets detection, auth/authz flaws, injection vectors, and infrastructure misconfigurations."
version: 1.0.0
---

# Security Audit

Perform a comprehensive security audit of a repository based on industry-standard methodologies (OWASP, SANS, CWE). Find vulnerabilities, misconfigurations, and security anti-patterns. Optionally create GitHub issues for critical findings.

---

## Step 0 — Determine target

If the user provided a GitHub URL, extract `owner/repo`.
If a local path, use directly.
If neither, use current working directory.

Ask: **"Should I create GitHub issues for critical/high findings?"** if not specified.

---

## Step 1 — Reconnaissance

Understand the application before attacking:

1. **Identify stack**: language, framework, database, cloud provider
2. **Map attack surface**: endpoints, auth mechanisms, file uploads, external integrations
3. **Read configs**: `docker-compose.yml`, `Dockerfile`, `.env.example`, CI/CD files, IaC templates
4. **Identify entry points**: API routes, CLI commands, event handlers, WebSocket endpoints
5. **Check dependencies**: `requirements.txt`, `package.json`, `Cargo.toml`, `go.mod`

Output: brief summary of stack, architecture, and attack surface.

---

## Step 2 — Dependency Analysis

### 2.1 Known Vulnerabilities

```bash
# Python
gh api repos/<owner>/<repo>/dependabot/alerts --jq '.[] | select(.state=="open")'
# Or locally:
pip audit / safety check / poetry show --outdated

# Node
npm audit --json / yarn audit --json

# Go
govulncheck ./...
```

### 2.2 Supply Chain Risks

Check for:
- Typosquatting packages (names similar to popular packages)
- Pinned vs unpinned versions
- Lock file presence and integrity
- Private registry configs exposed
- Post-install scripts in dependencies

---

## Step 3 — Secrets & Sensitive Data

Scan for hardcoded secrets:

### Patterns to search

| Type | Regex Pattern |
|------|--------------|
| AWS Keys | `AKIA[0-9A-Z]{16}` |
| Generic API Key | `[aA][pP][iI][-_]?[kK][eE][yY].*['"][0-9a-zA-Z]{20,}['"]` |
| JWT Secret | `(jwt|JWT|secret|SECRET).*['"][^'"]{8,}['"]` |
| Private Key | `-----BEGIN (RSA|EC|DSA|OPENSSH) PRIVATE KEY-----` |
| Password in code | `(password|passwd|pwd)\s*=\s*['"][^'"]+['"]` |
| Connection strings | `(mongodb|mysql|postgres|redis)://[^\s'"]+` |
| Tokens | `(token|TOKEN)\s*=\s*['"][0-9a-zA-Z\-_.]{20,}['"]` |
| Stripe keys | `sk_(live|test)_[0-9a-zA-Z]{24,}` |
| GitHub tokens | `gh[pousr]_[A-Za-z0-9_]{36,}` |

### Files to check
- `.env`, `.env.*` (should be in `.gitignore`)
- Config files committed to repo
- Test fixtures with real credentials
- CI/CD pipeline files (secrets in plain text)
- Dockerfile with ARG/ENV secrets
- `git log` for previously committed secrets

---

## Step 4 — Authentication & Session Management

### 4.1 Auth Flaws

| Vulnerability | What to look for |
|---------------|-----------------|
| Weak password policy | No min length, no complexity, no bcrypt/argon2 |
| Missing rate limiting | Login/signup/reset without throttle |
| Token issues | No expiration, weak secret, algorithm confusion (none/HS256) |
| Session fixation | Session ID not rotated after login |
| Credential stuffing | No account lockout, no CAPTCHA |
| Email enumeration | Different responses for existing/non-existing users |
| Password reset flaws | Predictable tokens, no expiration, token reuse |
| OAuth misconfig | Missing state param, open redirect in callback |
| 2FA bypass | Backup codes predictable, no brute-force protection |

### 4.2 JWT Specific

- Algorithm set to `none` accepted?
- HMAC key brute-forceable (short/common)?
- No `exp` claim?
- Secret in source code?
- `kid` parameter injection?
- JWK/JWKS injection?

---

## Step 5 — Authorization & Access Control

| Vulnerability | What to look for |
|---------------|-----------------|
| IDOR | Sequential IDs in URLs without ownership check |
| Privilege escalation | Role checks missing on sensitive endpoints |
| Broken object-level auth | User A accessing User B resources |
| Missing function-level auth | Admin endpoints without role guard |
| Mass assignment | Accepting `role`, `is_admin`, `credits` from request body |
| Path traversal | `../` in file paths not sanitized |
| Forced browsing | Hidden endpoints accessible without auth |

### Code patterns to flag

```python
# IDOR - no ownership check
@app.get("/users/{user_id}/data")
def get_data(user_id: int):
    return db.query(User).get(user_id)  # WHO is requesting?

# Mass assignment
user.update(**request.json)  # Accepts ANY field including role
```

---

## Step 6 — Injection Attacks

### 6.1 SQL Injection

```python
# VULNERABLE - string formatting
query = f"SELECT * FROM users WHERE id = {user_input}"
cursor.execute(f"DELETE FROM {table} WHERE id = %s")  # table name injection

# SAFE - parameterized
cursor.execute("SELECT * FROM users WHERE id = %s", (user_input,))
```

Look for: raw SQL with string interpolation, ORM `.extra()`, `.raw()`, dynamic table/column names.

### 6.2 Command Injection

```python
# VULNERABLE
os.system(f"ping {user_input}")
subprocess.call(f"convert {filename} output.png", shell=True)

# SAFE
subprocess.run(["ping", user_input], shell=False)
```

Look for: `os.system`, `subprocess` with `shell=True`, `eval()`, `exec()`, backticks in any language.

### 6.3 Template Injection (SSTI)

```python
# VULNERABLE - Jinja2
template = Template(user_input)  # User controls template
render_template_string(user_input)

# SAFE
render_template("fixed_template.html", data=user_input)
```

### 6.4 XSS (Cross-Site Scripting)

- `innerHTML`, `dangerouslySetInnerHTML`, `v-html`
- Unescaped template variables: `{!! $var !!}`, `<%- var %>`
- Reflected input in HTML without encoding
- DOM-based XSS via `document.location`, `window.name`

### 6.5 LDAP / NoSQL / GraphQL Injection

- MongoDB: `$where`, `$regex`, `$gt` from user input
- GraphQL: introspection enabled in prod, no depth/complexity limits
- LDAP: unescaped `(cn={input})` filters

---

## Step 7 — API Security

| Check | Description |
|-------|-------------|
| Rate limiting | Missing or bypassable (per-IP vs per-user) |
| Input validation | Missing max length, type coercion, nested depth |
| Error disclosure | Stack traces, SQL errors, internal paths in responses |
| CORS | `Access-Control-Allow-Origin: *` with credentials |
| Content-Type | No validation, accepts unexpected types |
| Pagination | No max limit (DoS via `?limit=999999`) |
| Batch/GraphQL | No query complexity limits |
| Versioning | Old API versions still active with known vulns |
| HTTPS | HTTP allowed, no HSTS header |
| CSP | Missing or `unsafe-inline`/`unsafe-eval` |

---

## Step 8 — Cryptography

| Vulnerability | What to look for |
|---------------|-----------------|
| Weak hashing | MD5, SHA1 for passwords (use bcrypt/argon2/scrypt) |
| ECB mode | AES-ECB (use CBC/GCM with random IV) |
| Hardcoded keys | Encryption keys in source code |
| Weak randomness | `Math.random()`, `random.random()` for security (use `secrets`) |
| No salt | Hashing without unique salt per entry |
| Broken TLS | Accepting TLS 1.0/1.1, self-signed certs in prod |
| Key derivation | Direct use of password as key (use PBKDF2/Argon2) |

---

## Step 9 — Infrastructure & Configuration

### 9.1 Docker

- Running as root (`USER` directive missing)
- Secrets in build args
- `latest` tag (unpinned)
- Exposed debug ports
- No health check
- Writable filesystem when unnecessary

### 9.2 CI/CD

- Secrets in plain text in workflow files
- `pull_request_target` with checkout of PR code (code injection)
- No branch protection on main
- Artifacts with sensitive data
- Self-hosted runners without isolation

### 9.3 Cloud (AWS/GCP/Azure)

- S3 buckets public
- IAM policies with `*` resource
- Security groups with `0.0.0.0/0` on non-HTTP ports
- Lambda with excessive permissions
- No encryption at rest
- CloudTrail/audit logging disabled

### 9.4 Database

- Default credentials
- No connection encryption
- Exposed to internet
- No query timeout
- Missing indexes on auth tables (timing attacks)

---

## Step 10 — Business Logic

| Vulnerability | What to look for |
|---------------|-----------------|
| Race conditions | TOCTOU in balance/credit operations |
| Price manipulation | Client-side price sent to server |
| Flow bypass | Skip steps in multi-step process |
| Abuse of features | Unlimited free tier, referral abuse |
| Integer overflow | Negative quantities, MAX_INT amounts |
| Replay attacks | No idempotency keys on mutations |

---

## Step 11 — File Operations

| Vulnerability | What to look for |
|---------------|-----------------|
| Unrestricted upload | No extension/MIME validation, no size limit |
| Path traversal | `../` in filenames not stripped |
| Zip slip | Archive extraction without path validation |
| XXE | XML parsing with external entities enabled |
| SSRF | User-supplied URLs fetched server-side without allowlist |
| Symlink attacks | Following symlinks in temp/upload dirs |

---

## Step 12 — Scoring & Report

### Severity Classification (CVSS-inspired)

| Severity | Score | Criteria |
|----------|-------|----------|
| Critical | 9.0-10.0 | RCE, auth bypass, full data breach, supply chain |
| High | 7.0-8.9 | SQLi, stored XSS, IDOR on sensitive data, privilege escalation |
| Medium | 4.0-6.9 | CSRF, reflected XSS, info disclosure, missing rate limit |
| Low | 0.1-3.9 | Missing headers, verbose errors, minor misconfig |
| Info | 0.0 | Best practice not followed, no direct exploit |

### Output Format

For each finding:

```markdown
## [SEVERITY] Title

**Category:** OWASP A01-A10 / CWE-XXX
**Location:** `file:line`
**Impact:** What an attacker can do
**Proof:** Code snippet showing the vulnerability
**Fix:** Remediation with code example
**References:** Links to relevant docs/standards
```

---

## Step 13 — Final Report

```markdown
# Security Audit Report

**Repository:** owner/repo
**Date:** YYYY-MM-DD

## Executive Summary

- Critical: N
- High: N
- Medium: N
- Low: N
- Total findings: N

## Findings

[List all findings sorted by severity]

## Recommendations

[Prioritized action items]

## Methodology

Based on:
- OWASP Testing Guide v4.2
- OWASP Top 10 (2021)
- CWE Top 25
- SANS Top 25
- PTES (Penetration Testing Execution Standard)
```

---

## Guidelines

- **Never exploit** — only identify and document
- **No false positives** — only flag code YOU can confirm is vulnerable
- **Context matters** — internal tools have different risk profiles than public APIs
- **Check git history** — secrets may have been committed and removed
- **Prioritize** — focus on what's exploitable, not theoretical
- **Be specific** — include file:line, code snippet, and concrete fix
- **Reference standards** — CWE, OWASP, CVE when applicable
- Run agents in parallel for Steps 3-11 when possible to maximize efficiency

---

## Quick Reference — OWASP Top 10 (2021)

| # | Category | Key checks |
|---|----------|-----------|
| A01 | Broken Access Control | IDOR, missing auth, CORS, path traversal |
| A02 | Cryptographic Failures | Weak hash, plaintext secrets, bad TLS |
| A03 | Injection | SQLi, XSS, command, template, LDAP |
| A04 | Insecure Design | Missing threat model, no rate limit by design |
| A05 | Security Misconfiguration | Debug on, default creds, unnecessary features |
| A06 | Vulnerable Components | Outdated deps, known CVEs, no updates |
| A07 | Auth Failures | Weak passwords, no MFA, credential stuffing |
| A08 | Software/Data Integrity | Deserialization, CI/CD tampering, unsigned updates |
| A09 | Logging Failures | No audit log, sensitive data in logs, no alerting |
| A10 | SSRF | Unvalidated URLs, internal service access |
