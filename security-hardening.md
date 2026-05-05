# Security Hardening Protocol

Full security audit and hardening guide. Read this entirely for any security-sensitive code.

---

## TABLE OF CONTENTS
1. Authentication & Authorization
2. Input Validation & Sanitization
3. Cryptography Standards
4. Database Security
5. API Security
6. File & Upload Security
7. Dependency Security
8. HTTP Security Headers
9. Secret Management
10. Logging & Audit Trails
11. OWASP Top 10 — Current Mitigations
12. Security Code Review Checklist

---

## 1. AUTHENTICATION & AUTHORIZATION

### Authentication Fundamentals
```
RULE: Authentication answers "who are you?" — Authorization answers "what can you do?"
These are SEPARATE concerns. Conflating them is a design defect.
```

**Password Handling (Non-Negotiable)**
- Never store plaintext passwords. Never.
- Use bcrypt (cost ≥ 12), Argon2id (recommended), or scrypt. NOT SHA-256. NOT MD5.
- Never log passwords, even in debug mode. Audit logs for `req.body` must redact password fields.
- Password reset tokens must be: cryptographically random, single-use, time-limited (≤15 mins), and hashed at rest.

**Session Management**
- Session IDs must be cryptographically random (≥128 bits of entropy)
- Rotate session ID on privilege escalation (login, sudo, role change)
- Set session expiry: idle timeout (15-30 min) AND absolute timeout (8-24 hrs)
- Session tokens in HttpOnly, Secure, SameSite=Strict cookies — NOT in localStorage
- Invalidate session on logout server-side, not just client-side

**JWT — When You Must Use It**
- Use asymmetric signing (RS256, ES256) for tokens used across services
- NEVER use algorithm=none. Reject it explicitly.
- Validate: signature, expiry (exp), not-before (nbf), issuer (iss), audience (aud)
- Keep expiry short (15-60 min). Use refresh tokens with rotation.
- Store refresh tokens in HttpOnly cookies, not localStorage
- Revocation requires a denylist or short expiry + rotation

**Authorization Patterns**
```python
# WRONG — Trusting client-supplied resource ID
def get_document(user_id, doc_id):
    return Document.find(doc_id)  # Anyone can access any document!

# CORRECT — Verify ownership at the data layer
def get_document(user_id, doc_id):
    doc = Document.find(doc_id)
    if doc.owner_id != user_id:
        raise PermissionDenied("Access denied")
    return doc
```

**Role-Based Access Control (RBAC)**
- Define roles as an enumeration, not strings compared with ==
- Check roles at the route middleware layer AND at the service layer
- Principle of Least Privilege: users get minimum permissions needed
- Never check `if user.role != 'banned'` — check `if user.can('action')` instead (allowlist, not blocklist)

---

## 2. INPUT VALIDATION & SANITIZATION

### Validation Hierarchy (All Four Layers)

```
Layer 1 — Type Validation:    Is it the right type? (string, int, bool)
Layer 2 — Format Validation:  Is it in the right format? (email regex, UUID format)
Layer 3 — Range Validation:   Is it in acceptable bounds? (age: 0-150, price: 0.01-999999)
Layer 4 — Business Validation: Is it valid in this context? (does this user ID exist?)
```

NEVER skip layers. A valid email format doesn't mean it's an email that belongs to this user.

### Sanitization vs Validation
- **Validation** REJECTS bad input (returns 400)
- **Sanitization** TRANSFORMS input to safe form (trim whitespace, normalize unicode)
- **Escaping** ENCODES output to prevent injection (HTML entities, SQL params)
- Do all three. They are not interchangeable.

### Injection Prevention Reference

**SQL Injection**
```python
# NEVER — String concatenation
cursor.execute(f"SELECT * FROM users WHERE name = '{user_input}'")

# ALWAYS — Parameterized queries
cursor.execute("SELECT * FROM users WHERE name = %s", (user_input,))

# ORM (still verify it generates parameterized queries)
User.where(name: user_input).first
```

**NoSQL Injection (MongoDB)**
```javascript
// NEVER — Object injection
db.users.find({ username: req.body.username })  // attacker sends { $gt: "" }

// ALWAYS — Explicit field typing
const username = String(req.body.username)
db.users.find({ username: username })
```

**Command Injection**
```python
# NEVER
os.system(f"convert {filename} output.pdf")

# ALWAYS — Use subprocess with list args, never shell=True
subprocess.run(["convert", filename, "output.pdf"], shell=False)
```

**LDAP Injection**
- Escape special characters: `( ) * \ / NUL` in LDAP queries
- Use LDAP library's parameterized search methods

**XSS Prevention**
```javascript
// NEVER — Direct DOM injection
element.innerHTML = userContent;

// ALWAYS — Text content or sanitized HTML
element.textContent = userContent;                              // Plain text
element.innerHTML = DOMPurify.sanitize(userContent, config);   // Rich text with sanitization

// React: JSX auto-escapes — but be careful with dangerouslySetInnerHTML
// Django: template auto-escapes — but mark_safe() defeats this
// Never use mark_safe() with user-controlled data
```

**Server-Side Template Injection (SSTI)**
```python
# NEVER — User input in template strings
Template(user_input).render(context)

# ALWAYS — Pass as variable, not as template
Template("Hello {{ name }}").render({"name": user_input})
```

---

## 3. CRYPTOGRAPHY STANDARDS

### Algorithm Reference (2024+)

| Purpose | Use | Never Use |
|---|---|---|
| Password hashing | Argon2id, bcrypt (≥12 rounds) | MD5, SHA-1, SHA-256, SHA-512 (raw) |
| Data encryption | AES-256-GCM, ChaCha20-Poly1305 | DES, 3DES, RC4, AES-ECB |
| Asymmetric encryption | RSA-OAEP (4096-bit), X25519 | RSA PKCS#1 v1.5, small key sizes |
| Digital signatures | Ed25519, ECDSA (P-256), RSA-PSS | RSA PKCS#1 v1.5 |
| Key derivation | Argon2id, PBKDF2 (600K rounds+) | Old PBKDF2 (<100K rounds), no KDF |
| HMAC | HMAC-SHA256, HMAC-SHA512 | HMAC-MD5, HMAC-SHA1 |
| Token generation | `secrets.token_urlsafe()`, `crypto.randomBytes()` | `Math.random()`, `random.random()` |
| TLS | TLS 1.2 minimum, TLS 1.3 preferred | SSL, TLS 1.0, TLS 1.1 |

### Critical Cryptographic Mistakes

**Never Roll Your Own Crypto**
Using a standard library's AES in ECB mode is still rolling your own. Use authenticated encryption (AEAD).
`AES-GCM` authenticates AND encrypts. Raw `AES-CBC` does not authenticate.

**IV/Nonce Reuse Is Fatal**
In AES-GCM, reusing a nonce with the same key completely breaks confidentiality.
Always generate a fresh random nonce for every encryption. Never use a counter without careful implementation.

**Timing Safe Comparisons**
```python
# NEVER — Leaks token length and value through timing
if token == stored_token:

# ALWAYS — Constant time
import hmac
if hmac.compare_digest(token.encode(), stored_token.encode()):
```

**Random Number Generation**
```python
# NEVER — Predictable PRNG for security purposes
import random
token = random.randint(100000, 999999)

# ALWAYS — CSPRNG
import secrets
token = secrets.token_hex(32)
```

---

## 4. DATABASE SECURITY

**Connection Security**
- Use TLS for all database connections, even internal
- Store connection strings in environment variables, never code
- Use connection pooling — never create connections per request
- Minimum privilege: app user should have only SELECT/INSERT/UPDATE on needed tables
- Separate read and write users if your DB supports it

**Query Hardening**
```sql
-- Stored procedures can reduce attack surface (but aren't magic)
-- Use them for complex multi-step operations

-- Always use LIMIT on user-facing queries
SELECT * FROM products WHERE category = ? LIMIT 100 OFFSET ?

-- Avoid SELECT * in production code — be explicit about columns
SELECT id, name, price FROM products WHERE category = ?
```

**Sensitive Data at Rest**
- Encrypt PII columns if breach is a realistic threat model
- Use column-level encryption or field-level encryption for: SSN, payment data, health data
- Encryption at the DB layer (TDE) protects against disk theft, NOT against SQL injection
- Consider tokenization for payment data (PCI-DSS) — don't store raw card numbers

**Migration Security**
- Never run migrations as root/superuser in production
- Migrations must be reversible where possible
- Test migrations on a copy of production data before running
- Lock the table / plan for downtime on large migrations

---

## 5. API SECURITY

**Input Validation at the Border**
Every API endpoint is an untrusted boundary. Treat all input as malicious until proven safe.

```javascript
// Validation schema example (using Zod/Joi/Yup pattern)
const createUserSchema = z.object({
  email: z.string().email().max(255),
  username: z.string().min(3).max(50).regex(/^[a-zA-Z0-9_-]+$/),
  age: z.number().int().min(13).max(120),
  // DO NOT include: isAdmin, role, createdAt — these are server-set
})
```

**Rate Limiting Strategy**
```
Public endpoints:       100 req/min per IP
Authenticated endpoints: 1000 req/min per user
Auth endpoints:          5 req/min per IP (login, register, password reset)
OTP/2FA verification:    3 attempts, then 15-min lockout
```

**CORS Configuration**
```javascript
// NEVER — Wildcard in production
Access-Control-Allow-Origin: *

// ALWAYS — Explicit allowlist
const allowedOrigins = ['https://app.example.com', 'https://admin.example.com']
if (allowedOrigins.includes(req.headers.origin)) {
  res.setHeader('Access-Control-Allow-Origin', req.headers.origin)
}
```

**Webhook Security**
- Verify webhook signatures — don't trust IP alone
- Use HMAC-SHA256 of the raw body with a shared secret
- Reject replays older than 5 minutes (include timestamp in payload)

---

## 6. FILE & UPLOAD SECURITY

**Path Traversal Prevention**
```python
import os

def safe_file_path(base_dir: str, user_filename: str) -> str:
    # Normalize and resolve absolute path
    safe_base = os.path.realpath(base_dir)
    full_path = os.path.realpath(os.path.join(safe_base, user_filename))
    
    # Ensure the resolved path is WITHIN the base directory
    if not full_path.startswith(safe_base + os.sep):
        raise SecurityError("Path traversal detected")
    
    return full_path
```

**Upload Security Checklist**
- [ ] Validate MIME type server-side (not client Content-Type header — that's spoofable)
- [ ] Validate file extension (allowlist, not blocklist)
- [ ] Re-encode images through a library (PIL/Pillow) to strip embedded payloads
- [ ] Limit file size (per file AND per user per day)
- [ ] Store uploads OUTSIDE the web root — serve through a signed URL or proxy
- [ ] Generate new random filenames — never use user-supplied filenames
- [ ] Scan uploads with antivirus for user-uploaded executables
- [ ] Never serve uploaded files with execute permissions

---

## 7. DEPENDENCY SECURITY

**Dependency Management Rules**
1. Pin all dependency versions in production (lockfiles are mandatory)
2. Audit dependencies before adding: `npm audit`, `pip-audit`, `cargo audit`
3. Minimize dependency count — every dependency is an attack surface
4. Monitor for CVEs: Dependabot, Snyk, or equivalents
5. Never use packages with <1000 weekly downloads for critical functions
6. Verify package integrity: `npm ci` uses lockfile, not `npm install`

**Supply Chain Attack Prevention**
- Verify package authors and download counts before use
- Pin to specific version hashes, not ranges (`1.2.3` not `^1.2.3` for security-sensitive deps)
- Use private registries for internal packages
- Enable two-factor on all package registry accounts

---

## 8. HTTP SECURITY HEADERS

Every web application must serve these headers:

```
Content-Security-Policy: default-src 'self'; script-src 'self'; object-src 'none';
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
X-Frame-Options: DENY
X-Content-Type-Options: nosniff
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: camera=(), microphone=(), geolocation=()
```

**CSP Notes**
- `'unsafe-inline'` for scripts defeats most XSS protection
- Use nonces or hashes instead of `'unsafe-inline'`
- Test CSP with report-only mode first: `Content-Security-Policy-Report-Only`

---

## 9. SECRET MANAGEMENT

**Secret Hierarchy**
```
Environment Variables  →  for local dev (never committed)
Secret Manager         →  for production (AWS Secrets Manager, HashiCorp Vault, GCP Secret Manager)
Encrypted Config       →  for edge cases (encrypted with KMS, decrypted at runtime)
```

**Never**
- Secrets in source code
- Secrets in version control (even private repos — rotate immediately if this happens)
- Secrets in log output
- Secrets in environment variables that get printed during startup
- Secrets in Docker image layers
- Secrets in CI/CD environment variables that get printed in logs

**Secret Rotation**
- Build your system to support secret rotation from day one
- No secret should be older than 90 days in production
- Database credentials, API keys, and certificates must all be rotatable without downtime

---

## 10. LOGGING & AUDIT TRAILS

**What to Log**
```
Authentication events: login success, login failure, logout, token refresh
Authorization failures: access denied, privilege escalation attempts
Data mutations: create, update, delete on sensitive resources
Security events: password change, email change, MFA changes
Admin actions: all admin operations with before/after state
```

**What Never to Log**
```
Passwords (even hashed)
Full credit card numbers (log last 4 only)
Full SSNs (log last 4 only)
Session tokens or JWTs
Private keys or secrets
Full request bodies without redaction for sensitive fields
```

**Structured Logging**
```json
{
  "timestamp": "2024-01-15T10:30:00Z",
  "level": "WARN",
  "event": "auth.login.failed",
  "userId": null,
  "ip": "203.0.113.42",
  "userAgent": "Mozilla/5.0...",
  "attemptedEmail": "user@example.com",
  "reason": "invalid_password",
  "attemptCount": 3
}
```

---

## 11. OWASP TOP 10 — CURRENT MITIGATIONS

| Rank | Vulnerability | Key Mitigation |
|---|---|---|
| A01 | Broken Access Control | Ownership checks at data layer, deny by default |
| A02 | Cryptographic Failures | Argon2id passwords, AES-256-GCM encryption, TLS 1.3 |
| A03 | Injection | Parameterized queries, input validation, output escaping |
| A04 | Insecure Design | Threat modeling, security in design phase, not afterthought |
| A05 | Security Misconfiguration | Hardened defaults, disable unnecessary features, audit configs |
| A06 | Vulnerable Components | Dependency scanning, pin versions, monitor CVEs |
| A07 | Auth Failures | MFA, rate limiting, session rotation, secure password storage |
| A08 | Integrity Failures | Signed packages, HMAC webhook verification, SRI for CDN |
| A09 | Logging Failures | Structured logs, no secrets in logs, tamper-evident audit trails |
| A10 | SSRF | Allowlist outbound requests, block private IP ranges |

---

## 12. SECURITY CODE REVIEW CHECKLIST

Run this on every PR that touches auth, data access, file handling, or external I/O:

**Authentication & Sessions**
- [ ] Passwords hashed with Argon2id or bcrypt (≥12 rounds)?
- [ ] Session tokens cryptographically random?
- [ ] Session rotated on privilege change?
- [ ] Sensitive cookies: HttpOnly + Secure + SameSite?
- [ ] JWT algorithm validated and not `none`?

**Authorization**
- [ ] Every route has explicit auth middleware?
- [ ] Every resource access verifies ownership?
- [ ] Admin-only routes are protected at multiple layers?

**Input Handling**
- [ ] All user inputs validated (type, length, format, range)?
- [ ] SQL uses parameterized queries exclusively?
- [ ] HTML output escapes user-controlled data?
- [ ] File paths are canonicalized and bounded?

**Cryptography**
- [ ] Only approved algorithms from the reference table?
- [ ] All tokens generated with CSPRNG?
- [ ] Secret comparisons use constant-time functions?

**Data**
- [ ] No secrets in code or logs?
- [ ] PII handled according to policy?
- [ ] Sensitive fields redacted in logs?

**Infrastructure**
- [ ] Security headers present?
- [ ] Rate limiting on auth endpoints?
- [ ] Dependencies audited for CVEs?
