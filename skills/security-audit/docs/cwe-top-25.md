# CWE Top 25 Most Dangerous Software Weaknesses (2023)

Quick reference for the most common weaknesses found in software.

| Rank | CWE ID | Name | Category |
|------|--------|------|----------|
| 1 | CWE-787 | Out-of-bounds Write | Memory |
| 2 | CWE-79 | Cross-site Scripting (XSS) | Injection |
| 3 | CWE-89 | SQL Injection | Injection |
| 4 | CWE-416 | Use After Free | Memory |
| 5 | CWE-78 | OS Command Injection | Injection |
| 6 | CWE-20 | Improper Input Validation | Validation |
| 7 | CWE-125 | Out-of-bounds Read | Memory |
| 8 | CWE-22 | Path Traversal | Access Control |
| 9 | CWE-352 | Cross-Site Request Forgery (CSRF) | Access Control |
| 10 | CWE-434 | Unrestricted File Upload | Injection |
| 11 | CWE-862 | Missing Authorization | Access Control |
| 12 | CWE-476 | NULL Pointer Dereference | Memory |
| 13 | CWE-287 | Improper Authentication | Auth |
| 14 | CWE-190 | Integer Overflow | Memory |
| 15 | CWE-502 | Deserialization of Untrusted Data | Injection |
| 16 | CWE-77 | Command Injection | Injection |
| 17 | CWE-119 | Improper Buffer Operations | Memory |
| 18 | CWE-798 | Hardcoded Credentials | Auth |
| 19 | CWE-918 | Server-Side Request Forgery (SSRF) | Access Control |
| 20 | CWE-306 | Missing Authentication for Critical Function | Auth |
| 21 | CWE-362 | Race Condition | Concurrency |
| 22 | CWE-269 | Improper Privilege Management | Access Control |
| 23 | CWE-94 | Code Injection | Injection |
| 24 | CWE-863 | Incorrect Authorization | Access Control |
| 25 | CWE-276 | Incorrect Default Permissions | Access Control |

---

## By Category

### Injection (8)
- CWE-79: XSS
- CWE-89: SQL Injection
- CWE-78: OS Command Injection
- CWE-434: File Upload
- CWE-502: Deserialization
- CWE-77: Command Injection
- CWE-94: Code Injection
- CWE-20: Input Validation

### Access Control (7)
- CWE-22: Path Traversal
- CWE-352: CSRF
- CWE-862: Missing Authorization
- CWE-918: SSRF
- CWE-269: Privilege Management
- CWE-863: Incorrect Authorization
- CWE-276: Default Permissions

### Authentication (3)
- CWE-287: Improper Authentication
- CWE-798: Hardcoded Credentials
- CWE-306: Missing Auth for Critical Function

### Memory Safety (6)
- CWE-787: Out-of-bounds Write
- CWE-416: Use After Free
- CWE-125: Out-of-bounds Read
- CWE-476: NULL Pointer Dereference
- CWE-190: Integer Overflow
- CWE-119: Buffer Operations

### Concurrency (1)
- CWE-362: Race Condition

---

## Language-Specific Relevance

### Python / JavaScript (Web Apps)
Most relevant: CWE-79, CWE-89, CWE-78, CWE-22, CWE-352, CWE-434, CWE-862, CWE-287, CWE-502, CWE-918, CWE-798, CWE-306, CWE-94, CWE-362

### Rust / C / C++ (Systems)
Most relevant: CWE-787, CWE-416, CWE-125, CWE-476, CWE-190, CWE-119, CWE-362

### Go (Backend/Infra)
Most relevant: CWE-89, CWE-78, CWE-22, CWE-862, CWE-287, CWE-918, CWE-798, CWE-362
