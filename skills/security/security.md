# Skill: Security

Application security best practices, patterns, and checklists.

---

## Secure Development Lifecycle

1. **Requirements**: identify security requirements and compliance obligations.
2. **Design**: threat modelling (STRIDE), architecture review.
3. **Implementation**: secure coding standards, code reviews.
4. **Testing**: SAST, DAST, SCA, manual pen test.
5. **Deployment**: secrets management, infrastructure security.
6. **Operations**: monitoring, incident response, patch management.

---

## Input Validation

**Validate at every trust boundary** — every place where data enters from an external source.

```typescript
// Use zod for strict runtime validation
import { z } from 'zod'

const CreateUserSchema = z.object({
  email: z.string().email().max(255).toLowerCase(),
  name: z.string().min(1).max(100).regex(/^[\p{L}\p{N} '-]+$/u),
  role: z.enum(['user', 'moderator']),  // whitelist, never accept arbitrary roles
  age: z.number().int().min(13).max(120).optional(),
})

// Validate at the API boundary; reject early
const result = CreateUserSchema.safeParse(req.body)
if (!result.success) {
  return res.status(400).json({ errors: result.error.flatten() })
}
```

---

## SQL Injection Prevention

```python
# BAD — string interpolation
cursor.execute(f"SELECT * FROM users WHERE email = '{email}'")

# BAD — string concatenation
query = "SELECT * FROM users WHERE id = " + user_id

# GOOD — parameterised query
cursor.execute("SELECT * FROM users WHERE email = %s", (email,))

# GOOD — ORM with parameterised queries
User.objects.filter(email=email)
```

---

## XSS Prevention

```typescript
// BAD — raw HTML injection
element.innerHTML = userInput

// GOOD — text content only
element.textContent = userInput

// GOOD — sanitise with DOMPurify when HTML is required
import DOMPurify from 'dompurify'
element.innerHTML = DOMPurify.sanitize(userInput)
```

Content Security Policy header:
```
Content-Security-Policy: 
  default-src 'self';
  script-src 'self' 'nonce-{random}';
  style-src 'self' 'nonce-{random}';
  img-src 'self' data: https:;
  connect-src 'self' https://api.example.com;
  frame-ancestors 'none';
```

---

## Authentication Patterns

### Password Hashing
```python
import bcrypt

# Hash on registration
hashed = bcrypt.hashpw(password.encode(), bcrypt.gensalt(rounds=12))

# Verify on login
is_valid = bcrypt.checkpw(password.encode(), hashed)
```

### JWT Best Practices
```typescript
// Short-lived access tokens (15 min) + rotating refresh tokens
const accessToken = jwt.sign(
  { sub: userId, role: user.role },
  process.env.JWT_SECRET!,
  { expiresIn: '15m', algorithm: 'HS256' }
)

// Validate on every request
const payload = jwt.verify(token, process.env.JWT_SECRET!, {
  algorithms: ['HS256'],
  issuer: 'https://auth.example.com',
  audience: 'https://api.example.com',
})
```

### Secure Cookie Settings
```typescript
res.cookie('session', token, {
  httpOnly: true,     // not accessible from JS
  secure: true,       // HTTPS only
  sameSite: 'strict', // CSRF protection
  maxAge: 7 * 24 * 60 * 60 * 1000, // 7 days
  path: '/',
})
```

---

## Secrets Management

Never store secrets in:
- Source code
- Docker images
- Environment variable files committed to git
- Log output
- Error messages

```typescript
// BAD
const STRIPE_KEY = "sk-prod-abc123"

// GOOD — from environment (loaded from a secrets manager at startup)
const STRIPE_KEY = process.env.STRIPE_KEY
if (!STRIPE_KEY) throw new Error("STRIPE_KEY environment variable is required")
```

Use a secrets manager:
- AWS Secrets Manager / Parameter Store
- HashiCorp Vault
- GCP Secret Manager
- Azure Key Vault

---

## Security Headers

```typescript
// Helmet.js (Express)
import helmet from 'helmet'

app.use(helmet({
  contentSecurityPolicy: { /* ... */ },
  hsts: { maxAge: 31536000, includeSubDomains: true, preload: true },
  referrerPolicy: { policy: 'strict-origin-when-cross-origin' },
  permittedCrossDomainPolicies: false,
  crossOriginEmbedderPolicy: true,
  crossOriginOpenerPolicy: { policy: 'same-origin' },
  crossOriginResourcePolicy: { policy: 'same-origin' },
}))
```

---

## SSRF Prevention

```typescript
import { isPrivateIP } from 'range_check'
import { URL } from 'url'

async function fetchUrl(rawUrl: string): Promise<Response> {
  const url = new URL(rawUrl)  // throws on invalid URL

  // Allowlist of permitted hosts
  const allowedHosts = ['api.github.com', 'hooks.slack.com']
  if (!allowedHosts.includes(url.hostname)) {
    throw new SecurityError(`Host ${url.hostname} is not allowed`)
  }

  // Block private/internal IPs (resolved after DNS lookup)
  const resolved = await dns.lookup(url.hostname)
  if (isPrivateIP(resolved.address)) {
    throw new SecurityError('Requests to private IP addresses are not allowed')
  }

  return fetch(rawUrl, { redirect: 'manual' })  // don't follow redirects
}
```

---

## Dependency Security

```yaml
# GitHub Actions — automated dependency scanning
- uses: aquasecurity/trivy-action@master
  with:
    scan-type: 'fs'
    severity: 'HIGH,CRITICAL'
    exit-code: '1'

# Dependabot — automated dependency updates
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: npm
    directory: /
    schedule:
      interval: weekly
    open-pull-requests-limit: 10
```

---

## Secret Scanning (pre-commit)

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.0
    hooks:
      - id: gitleaks
```

---

## Security Checklist

### Code Review
- [ ] No SQL string interpolation; parameterised queries used everywhere.
- [ ] All inputs validated at API boundaries.
- [ ] No secrets in source code.
- [ ] Authentication enforced on protected endpoints.
- [ ] Authorisation checked server-side (not just client-side).
- [ ] HTML output sanitised (DOMPurify or template engine auto-escaping).
- [ ] No `dangerouslySetInnerHTML` without sanitisation.

### Infrastructure
- [ ] TLS 1.2+ enforced; HTTP redirects to HTTPS.
- [ ] Security headers set (HSTS, CSP, X-Frame-Options).
- [ ] Least-privilege IAM roles for all service accounts.
- [ ] Secrets in a secrets manager, not in environment files or config maps.
- [ ] Network access restricted to required paths only.

### CI/CD
- [ ] SAST scan (Semgrep/CodeQL) with no HIGH/CRITICAL findings.
- [ ] SCA scan (Trivy/Snyk) with no CRITICAL CVEs.
- [ ] Secret scanning (Gitleaks) in CI.
- [ ] Container image scan with no CRITICAL vulnerabilities.
