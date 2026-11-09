# Security Policy

> Security is not a feature — it is a system property built from day one. This document
> defines the mandatory practices.
> Any deviation must be explicitly approved by the Tech Lead.

---

## Security principles

1. **Defense in Depth:** Multiple security layers. If one fails, the others contain the damage.
2. **Least Privilege:** Each component has only the minimum necessary permissions.
3. **Fail Secure:** In case of error, the system denies access, does not allow it.
4. **Security by Design:** Security controls are designed from the start, not added at the end.
5. **Zero Trust:** Always verify, never implicitly trust, even within the internal network.

---

## Authentication

> **Status (2026-10-23):** current behavior, including the security closure merged through the
> project's 2026-10 pull requests. Contract details: `07-api/authentication.md`.

### Opaque sessions (current, ADR-023)

- Login issues an **opaque session id** (`identity.user_session`), sent as
  `Authorization: Bearer <sessionId>`. Sessions last 480 minutes; `POST /api/v1/auth/refresh`
  rotates the id (single use: the previous row is closed). There is no JWT and no refresh token.
- **Edge check:** Kong's global `pre-function` requires an active session on every non-public
  route by introspecting identity (`GET /api/v1/sessions/{id}`).
- **Service check:** every service validates the bearer again and applies RBAC. The Java
  services (01/02/04/05) use the shared `back-end/shared/auth-java` library and the Node
  services (03/07/08/09) use `back-end/shared/auth-node`; each keeps its own permission rules. The libraries also hold the caller-school lookup and the role names (ADR-038).
  Biometric (06) uses its own `require_permission`. Only positive answers are cached, for 10 s.

### Login hardening

| Control | Rule |
|---|---|
| No default accounts | No changeset ships a password. The first administrator is created by hand with `database/scripts/bootstrap-admin.sh`; the old seed accounts are disabled by a changeset. |
| First-login password change | The bootstrap administrator, and accounts whose temporary credentials identity e-mailed, have `must_change_password`. Until it is changed, Kong rejects every write except the password change (403 `PASSWORD_CHANGE_REQUIRED`); web and mobile show the forced-change screen. |
| Brute force | Identity answers **429** with `Retry-After` from the 11th failed login per client IP within 60 s (on top of the per-account lockout). Successful logins do not count. |
| Passwords | bcrypt, cost 12 (ADR-008). |

### Browser access

- CORS allows only the origins listed in `KONG_CORS_ORIGINS` (each validated at startup); it is
  required in production. Native mobile requests send no `Origin` and are not affected.
- The web keeps the session id in `sessionStorage` (cleared when the tab closes), never in
  `localStorage`.

### Future evolution (ADR-007): JWT

> Target state only; nothing below is implemented.

| Property | Required value (target state) |
|----------|---------------|
| Signing algorithm | RS256 (asymmetric) or HS256 with 256+ bit secret |
| Access token expiration | 1 hour (`exp`) |
| Refresh token expiration | 7 days |
| Required claims | `sub` (userId), `iat`, `exp`, `jti` (unique token ID) |
| Client storage | `httpOnly cookie` (web) or Keychain/Keystore (mobile) |

Prohibited in the payload: passwords, card data, full PII (only the user ID). Refresh tokens
would be stored hashed, rotated on every use, invalidated on logout and password change, and
all of a user's tokens revoked if a revoked one is replayed.

---

## Authorization

### RBAC (Role-Based Access Control) — FaceAttend EDU

| Role (02-domain) | Description | Example permissions |
|------|-------------|------------|
| `Administrator` | Business administrator (global) | `users:manage`, `roles:assign`, `academic:manage`, `configuration:manage`, `justification:review` |
| `Instructor` / `Teacher` | Manages cohort attendance and reviews justifications | `attendance:record`, `justification:review`, `reports:read`, `schedule:manage` |
| `Student` / `Learner` | Records own attendance and submits justifications | `attendance:self-record`, `justification:create`, `history:self-read` |

> **Mapping:** effective role catalog in the database is `Administrador | Instructor | Aprendiz`
> (changeset `auth-004-unify-mobile-roles`; legacy `SUPER_ADMIN`/`SCHOOL_ADMIN`/`INSTRUCTOR`/
> `STUDENT` are merged). Roles are **global** — `authorization.role` has no `school_id` and
> `User_Role` maps `user_id → role_id` only. Permissions are resolved per request through
> `GET /api/v1/auth/evaluate`, not through JWT claims (no tokens are issued today).

**Permission model:**

```
Permission: [resource]:[action]

Examples:
  orders:create
  orders:read
  orders:update
  orders:delete
  users:read
  reports:export
```

**Validation (current vs target):**
- Today: the API Gateway **authenticates at the edge** — a global `pre-function` requires a valid
  active session on every non-public route (plus `cors` + `rate-limiting`), while every service
  validates the opaque bearer again on its own (`AuthTokenFilter` in Java, an equivalent guard in
  TS/Python) and checks permissions
- Target (ADR-007): the API Gateway validates the JWT (signature and expiration)
- Each service validates the role permissions for the specific operation
- Target (ADR-007): roles travel as claim `roles: ["Administrador"]`

---

## Secure communication

### Transmission

- **HTTPS mandatory** in all environments except local
- TLS 1.2 minimum; TLS 1.3 recommended
- Certificates: Let's Encrypt issued by Caddy in production (ADR-022); Caddy's local CA for local TLS
- HSTS enabled in production

### Internal service-to-service communication

- mTLS for service-to-service communication in production (if possible with service mesh)
- Bearer token or internal API key for services that do not support mTLS

---

## Secret management

```
✗ NEVER in source code
✗ NEVER in committed .env
✗ NEVER in logs
✗ NEVER in client error messages
✓ Environment variables (injected by the orchestrator)
✓ Vault (HashiCorp Vault, AWS Secrets Manager, etc.)
✓ Kubernetes Secrets (encrypted with KMS)
```

**Secret rotation:**
- API keys: every 90 days
- TLS certificates: 60 days before expiration
- DB passwords: every 6 months or immediately if compromise is suspected

**Secrets the stack requires** (`docker compose` refuses to start without them; in production they
come from the `ENV_PROD_FILE` deployment secret):

| Variable | Purpose |
|---|---|
| `POSTGRES_PASSWORD` | database owner, used only by migrations |
| `<SERVICE>_DB_PASSWORD` (8) | one database login per service, limited to its own schema |
| `MONGO_USER` / `MONGO_PASSWORD` | MongoDB with authentication enabled |
| `MINIO_ROOT_USER` / `MINIO_ROOT_PASSWORD` / `MINIO_KMS_SECRET_KEY` | supporting-document storage and its encryption key |
| `PII_ENCRYPTION_KEY` / `PII_HASH_KEY` | encryption and blind indexes of personal data |
| `BIOMETRIC_ENCRYPTION_KEY` | encryption of biometric templates |
| `KONG_CORS_ORIGINS` | allowed browser origins |

Data ports (Postgres, MongoDB, Redis, Kafka) and Kong's admin API bind to `127.0.0.1` only; the
security group exposes 80/443, plus 22 from the operator's IP.

**Least privilege in the database:** each service connects as `<service>_app`, a member of a
`svc_<schema>` role with rights on its own schema only (`database/*/03-dcl`); migrations keep
using the owner. `database/scripts/verify-schema-isolation.sh` proves a cross-schema `SELECT`
is denied.

---

## Data protection (Ley 1581 de 2012)

Personal and biometric data follow `00-governance/privacy-and-consent.md`: versioned privacy
notice, prior consent recorded with date and version, optional biometric consent, revocation
with deletion of biometric templates, and AES-256-GCM encryption at rest (ADR-016).

---

## Input validation and sanitization

### General rules

1. **Never trust user input.** Validate at the edge (controller) before processing.
2. **Whitelist, not blacklist.** Define what is allowed, not only what is prohibited.
3. **Reject early.** If input is invalid, respond 400 and do not process further.

### SQL Injection — Prevention

```typescript
// ✗ VULNERABLE
const result = await db.query(`SELECT * FROM users WHERE email = '${userInput}'`);

// ✓ SAFE — always use prepared parameters
const result = await db.query('SELECT * FROM users WHERE email = $1', [userInput]);
```

### XSS — Prevention

```typescript
// ✗ VULNERABLE — rendering HTML without escaping
element.innerHTML = userProvidedContent;

// ✓ SAFE — use textContent or sanitize
element.textContent = userProvidedContent;
// or with library: DOMPurify.sanitize(userProvidedContent)
```

### Validation with Zod / Joi

```typescript
// Explicit validation schema in the controller — FaceAttend example
const RecordAttendanceSchema = z.object({
  classSessionId: z.string().uuid(),
  academicActorId: z.string().uuid(),
  captureMethod: z.enum(['FACIAL', 'MANUAL', 'IMPORT']),
  capturedAt: z.string().datetime(),
});

const CreateUserSchema = z.object({
  schoolId: z.number().int().positive(),
  documentNumber: z.string().min(5).max(20),
  email: z.string().email().max(255),
  role: z.enum(['Administrator', 'Instructor', 'Student']),
});
```

---

## OWASP Top 10 — Review checklist

| Vulnerability | Implemented control |
|---------------|-------------------|
| A01: Broken Access Control | RBAC + permission validation in each service |
| A02: Cryptographic Failures | TLS 1.2+, bcrypt for passwords, secrets in vault |
| A03: Injection | Prepared parameters in SQL, schema validation |
| A04: Insecure Design | Threat modeling in design, Security review |
| A05: Security Misconfiguration | IaC for configuration, review of defaults |
| A06: Vulnerable Components | Dependabot / Snyk for automatic updates |
| A07: Authentication Failures | JWT with rotation, brute-force protection |
| A08: Software Integrity Failures | Verify dependency checksums, SBOM |
| A09: Logging Failures | Logs without PII, centralized, with alerts |
| A10: SSRF | Whitelist of external URLs, do not follow redirects automatically |

---

## Audit and security logs

### Events that are ALWAYS recorded

```typescript
// Security events — store in a separate log, with retention > 1 year
const SECURITY_EVENTS = [
  'auth.login.success',
  'auth.login.failure',
  'auth.login.brute_force_detected',
  'auth.password.changed',
  'auth.token.revoked',
  'auth.unauthorized_access_attempt',
  'data.pii.accessed',
  'admin.role.changed',
  'admin.user.deleted',
];
```

**Required fields in security logs:**
- `userId` (or `ANONYMOUS` if not authenticated)
- `sourceIp`
- `action`
- `resource`
- `result` (SUCCESS / FAILURE)
- `timestamp`

---

## Vulnerability process

### What to do if you find a vulnerability

1. **Do not commit it to the public repo** or discuss it in open channels
2. Immediately notify the Tech Lead via a private channel
3. Create a private issue or a restricted repository issue
4. Severity is assigned (CVSS score or internal classification)
5. Remediated in the current sprint if critical, in the next sprint if high

### Remediation SLAs

| Severity | Remediation time |
|----------|----------------|
| Critical (CVSS 9-10) | 24 hours |
| High (CVSS 7-8.9) | 1 week |
| Medium (CVSS 4-6.9) | 1 month |
| Low (CVSS < 4) | Next security review |

---

## Correlations

- Security non-functional requirements → `04-requirements/non-functional.md`
- ADR on authentication → `05-architecture/decisions/`
- Security event observability → `13-operations/observability.md`
- RBAC implemented in → `09-microservices/services/02-authorization-service/`
