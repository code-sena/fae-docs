# ADR-008 — Password Hashing Algorithm

- **ID:** ADR-008
- **Date:** 2026-09-12
- **Status:** Proposed
- **Authors:** FaceAttend EDU Team

---

## Context

The system requires secure password storage for user authentication (NFR-004, `00-governance/security-policy.md`). The security policy currently proposes "bcrypt with cost factor ≥ 12, or Argon2id" without committing to one. The `.env.example` file already has `APP_AUTH_BCRYPT_ROUNDS=10`, which is inconsistent with the policy's "≥ 12" requirement.

This decision affects:
- The `identity-service` module (password creation on registration, password change, and password verification on login — auth is merged into identity-service per ADR-005).
- The `.env.example` configuration.
- Compliance with NFR-004 security requirements.

**Known constraints:**
- Password hashing must be slow enough to resist brute-force attacks.
- The algorithm must be widely supported in the chosen runtime (Node.js/TypeScript).
- Must align with OWASP recommendations (referenced in `security-policy.md`).

---

## Decision

**We decided:** Use **bcrypt with cost factor 12** for password hashing.

**Justification:**

1. **Maturity and audit:** bcrypt has been the industry standard for password hashing for over 20 years. It has been extensively analyzed and no practical attacks against properly configured bcrypt exist.
2. **Node.js ecosystem:** `bcrypt` and `bcryptjs` are the most widely used password hashing libraries in Node.js. `bcryptjs` is a pure JavaScript implementation (no native compilation issues) with identical API.
3. **Simplicity:** bcrypt requires only one parameter (cost factor). Argon2id requires tuning three parameters (memory, iterations, parallelism) which increases the risk of misconfiguration.
4. **OWASP alignment:** OWASP recommends bcrypt, scrypt, or Argon2id. bcrypt satisfies this requirement.
5. **Operational simplicity:** No native compilation required (with `bcryptjs`). Docker builds are simpler.

**Configuration:**
- Cost factor: **12** (aligns with `security-policy.md` requirement of "≥ 12"; `.env.example` must be updated from 10 to 12).
- Salt: automatically generated per bcrypt specification (16-byte random salt).
- Hash format: `$2b$12$...` (bcrypt standard).

**Why not Argon2id:**
- Argon2id is theoretically superior (memory-hard, resistant to GPU/ASIC attacks).
- However, it requires tuning three parameters (memory cost, time cost, parallelism) and the optimal values depend on the deployment hardware.
- For a 3-person team deploying to a single institution, bcrypt's simplicity and proven track record outweigh Argon2id's theoretical advantages.
- Argon2id can be adopted later if threat model changes (e.g., multi-tenant SaaS with higher attack surface).

---

## Evaluated alternatives

| Alternative | Pros | Cons | Reason for discarding |
|------------|------|------|-----------------------|
| **bcrypt, cost 12 (chosen)** | Industry standard; audited; simple configuration; native Node.js support; OWASP approved | Susceptible to ASIC/GPU attacks (theoretical at this scale); fixed memory cost | — (chosen) |
| Argon2id | Memory-hard; resistant to GPU/ASIC; Password Hashing Competition winner; OWASP recommended | Three parameters to tune; risk of misconfiguration; less mature Node.js bindings; requires native compilation (`argon2` package) | Operational complexity disproportionate to current threat model; team can adopt later if needed |
| scrypt | Memory-hard; used by Apple/iCloud | Less widely supported in Node.js than bcrypt; fewer audited implementations | Smaller ecosystem; less community scrutiny than bcrypt |
| PBKDF2-SHA256 | Simple; NIST recommended; built into many crypto libraries | Not memory-hard; faster to brute-force than bcrypt at same iteration count | Weaker than bcrypt for the same computational cost |

---

## Consequences

**Positive:**
- Well-understood security properties; no configuration surprises.
- `bcryptjs` (pure JS) avoids native compilation issues in Docker builds.
- `.env.example` cost factor updated to 12, consistent with `security-policy.md`.
- OWASP Top 10 A02 (Cryptographic Failures) is addressed for password storage.

**Negative / Trade-offs:**
- bcrypt is not memory-hard, making it theoretically vulnerable to GPU/ASIC attacks (not a practical concern at this scale).
- Cost factor 12 means ~250ms per hash on modern hardware — acceptable for login/registration, not for bulk operations.

**Impact on the system:**
- Affected modules: `identity-service` (password creation/change and verification — auth merged per ADR-005).
- Documents that must be updated:
  - `.env.example` — change `APP_AUTH_BCRYPT_ROUNDS=10` to `APP_AUTH_BCRYPT_ROUNDS=12`
  - `00-governance/security-policy.md` §Password management — confirm bcrypt as the chosen algorithm
  - `09-microservices/services/01-identity-service/data-model.md` — document password_hash column as bcrypt hash

---

## Risks

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|-----------|
| Cost factor too high causes login latency | Low | Medium | Cost 12 adds ~250ms to login — within NFR-001 P95 target for auth endpoints; can reduce to 11 if needed |
| Cost factor too low allows brute-force | Low | High | Cost 12 is above OWASP minimum; monitor failed login rates via audit-service |
| Future quantum computing threats | Very Low | High | Not addressable by choice of hash algorithm; out of current scope |

---

## References

- Security policy → `00-governance/security-policy.md` §Password management
- NFR-004 Security → `04-requirements/non-functional.md`
- Environment variables → `.env.example`
- OWASP Password Storage Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html
