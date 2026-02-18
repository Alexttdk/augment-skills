---
type: agent_requested
description: Activate when implementing auth/authz, handling user input, preventing OWASP Top 10 vulnerabilities, or performing a security code review on any layer of the stack.
---

# Secure Code Guardian

## When to Use
- Implementing authentication or authorization flows
- Handling any user-supplied input (forms, APIs, file uploads, URLs)
- Writing database queries or external command execution
- Security hardening existing code or reviewing a PR for vulnerabilities
- Setting up session management, JWT, or OAuth
- Configuring HTTP headers, CORS, rate limiting

---

## Threat-Model First

Before coding, answer three questions:
1. **What assets need protecting?** (data, sessions, admin functions)
2. **Who are the adversaries?** (anonymous users, authenticated users, insiders)
3. **What are the attack vectors?** (inputs, APIs, dependencies, infrastructure)

Then apply defence-in-depth: validate → sanitise → parameterise → encode → authorise → monitor.

---

## OWASP Top 10 (2021) — Detection & Prevention Cheatsheet

| # | Category | Detect | Prevent |
|---|----------|--------|---------|
| A01 | Broken Access Control | Missing ownership checks, IDOR | Server-side authz on every request; deny by default |
| A02 | Cryptographic Failures | Plaintext secrets, weak algos (MD5, SHA-1) | bcrypt/argon2 for passwords; AES-256-GCM for data; TLS everywhere |
| A03 | Injection | String concat in SQL/shell/LDAP | Parameterised queries; ORMs; `execFile` over `exec` |
| A04 | Insecure Design | No threat model, trust-by-default | Fail-safe defaults; principle of least privilege |
| A05 | Security Misconfiguration | Debug on in prod, default creds, verbose errors | Hardened defaults; disable unused features; security headers |
| A06 | Vulnerable Components | Outdated deps with known CVEs | `npm audit` / `pip-audit`; pin versions; dependency scanning in CI |
| A07 | Auth & Identification Failures | No lockout, weak passwords, exposed tokens | bcrypt(12+); lockout after 5 fails; short-lived JWTs |
| A08 | Software & Data Integrity | No signature verification on artifacts | Verify checksums; signed commits; CSP |
| A09 | Logging & Monitoring Failures | No security event logs, silent errors | Log auth events, rate-limit hits, access denied; alert on anomalies |
| A10 | SSRF | User-controlled URLs fetched server-side | Allowlist hosts/protocols; block internal IP ranges |

---

## Input Validation Rules

**Always validate on the server — never trust client-side checks alone.**

```typescript
import { z } from 'zod';

// Schema-first validation (Zod / Pydantic)
const UserSchema = z.object({
  email:    z.string().email().max(255),
  name:     z.string().min(1).max(100).regex(/^[\w\s-]+$/),
  role:     z.enum(['user', 'admin']).default('user'),
});

// File uploads: check type + size + magic bytes
const ALLOWED_MIME = ['image/jpeg', 'image/png'];
const MAX_BYTES = 5 * 1024 * 1024;
// verify actual magic bytes, not just extension
```

**Injection prevention matrix:**

| Vector | ❌ Bad | ✅ Good |
|--------|--------|---------|
| SQL | `SELECT * FROM u WHERE id=${id}` | Parameterised: `db.query('…WHERE id=$1',[id])` |
| Shell | `exec('convert ' + input)` | `execFile('convert', ['-resize', safe])` |
| Path | `path.join('/uploads', input)` | `path.basename` + resolve-and-check startsWith |
| HTML | `innerHTML = userText` | `textContent`; or `DOMPurify.sanitize()` |
| URL | fetch(userProvidedUrl) | Allowlist protocol + hostname before fetching |

---

## Authentication Patterns

```typescript
// Password hashing — bcrypt with 12+ rounds
const hash = await bcrypt.hash(password, 12);
const ok   = await bcrypt.compare(password, hash);

// Password policy minimum
const strong = /^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[@$!%*?&]).{12,}$/;

// JWT — short-lived access + rotating refresh
const access  = jwt.sign({ sub: userId, type: 'access' },  SECRET, { expiresIn: '15m' });
const refresh = jwt.sign({ sub: userId, type: 'refresh' }, SECRET, { expiresIn: '7d' });
// Verify token type in middleware to prevent refresh-token privilege escalation

// Account lockout
const MAX_FAILS = 5, LOCKOUT_MS = 15 * 60 * 1000;
// Increment redis counter on failure; block if >= MAX_FAILS
```

**JWT pitfalls to avoid:**
- Never accept `alg: none`
- Never store in `localStorage` (XSS risk) — use `httpOnly` cookie
- Always verify `type` claim to prevent refresh token reuse as access token
- Rotate refresh tokens on each use (refresh token rotation)

**OAuth 2.0 / OIDC best practices:**
- Use PKCE for all public clients (SPAs, mobile)
- Validate `state` parameter to prevent CSRF
- Validate `iss`, `aud`, `exp` claims on every ID token
- Never expose client_secret in frontend code

---

## XSS & CSRF Prevention

```typescript
// CSP via helmet (Express)
app.use(helmet.contentSecurityPolicy({
  directives: {
    defaultSrc: ["'self'"],
    scriptSrc:  ["'self'"],
    objectSrc:  ["'none'"],
    frameSrc:   ["'none'"],
  },
}));

// CSRF: SameSite cookies + double-submit token
res.cookie('session', token, {
  httpOnly: true, secure: true, sameSite: 'strict', maxAge: 900_000,
});
// For APIs also accept X-CSRF-Token header and compare to cookie value
```

---

## Security Headers Quick-Config

```typescript
app.use(helmet());   // Sets safe defaults for all headers below

// Key headers to verify are present:
// Strict-Transport-Security: max-age=31536000; includeSubDomains
// X-Frame-Options: DENY
// X-Content-Type-Options: nosniff
// Referrer-Policy: strict-origin-when-cross-origin
// Permissions-Policy: geolocation=(), microphone=(), camera=()
```

**Rate limiting:**

```typescript
import rateLimit from 'express-rate-limit';

app.use('/api/', rateLimit({ windowMs: 15*60*1000, max: 100 }));
// Auth endpoints — stricter
app.post('/api/login', rateLimit({ windowMs: 15*60*1000, max: 5,
  skipSuccessfulRequests: true }), loginHandler);
```

---

## Secure-by-Default Checklist

**MUST DO**
- [ ] Hash passwords with bcrypt/argon2 (never MD5/SHA-1/plaintext)
- [ ] Use parameterised queries for every DB interaction
- [ ] Validate + sanitise all user input (schema-first)
- [ ] Set `httpOnly`, `secure`, `sameSite=strict` on session cookies
- [ ] Apply rate limiting on auth & sensitive endpoints
- [ ] Set security headers (use helmet or equivalent)
- [ ] Store secrets in env vars / secret manager — never in code
- [ ] Check resource ownership on every data-access call (no IDOR)
- [ ] Log authentication events, access-denied, and anomalies
- [ ] Scan dependencies for CVEs in CI (`npm audit`, `trivy`, `snyk`)

**MUST NOT**
- [ ] Trust client-supplied `userId`, `role`, or `isAdmin` fields
- [ ] Expose stack traces or internal paths in error responses
- [ ] Use `eval`, `Function()`, or dynamic code execution on user data
- [ ] Hardcode secrets, API keys, or credentials in source code
- [ ] Disable TLS verification (`rejectUnauthorized: false`)
- [ ] Log passwords, tokens, or PII

---

## Code Review Security Workflow

1. **Inputs** — every entry point validated? types, length, format, allowlist?
2. **Queries** — parameterised? ORM used? no raw string concat?
3. **Auth** — every route requires authentication where expected?
4. **Authz** — resource ownership checked server-side for every mutation?
5. **Secrets** — no hardcoded credentials, keys, or connection strings?
6. **Output** — HTML output encoded? API responses exclude sensitive fields?
7. **Dependencies** — no known vulnerable packages?
8. **Headers** — CSP, HSTS, CSRF protection present?
9. **Logging** — security events logged? PII excluded from logs?
10. **Error handling** — generic errors to client, detailed to server logs?

## References
- Source: https://github.com/Jeffallan/claude-skills/tree/main/skills/secure-code-guardian
- OWASP Top 10 (2021): https://owasp.org/Top10/
- OWASP Cheat Sheet Series: https://cheatsheetseries.owasp.org/

