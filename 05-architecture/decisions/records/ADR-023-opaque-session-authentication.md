# ADR-023 — Opaque session authentication with edge introspection (instead of JWT)

- **ID:** ADR-023
- **Date:** 2026-10-23
- **Status:** Accepted
- **Decision (2026-10-23):** accepted; records the behaviour already running. JWT RS256 stays a deferred option (`AT-007`).
- **Authors:** FaceAttend EDU Team
- **Amended by ADR-038 (2026-11-08):** the "no shared library" constraint in the context no longer holds: the Java and Node services share the session check, the caller-school lookup and the role names through `back-end/shared/auth-java` and `back-end/shared/auth-node`. Biometric (Python) keeps its own check.
- **Implementation note (2026-10-23, verified against `FULL/` and `07-api/authentication.md`):** this is the behaviour already running; the ADR records it as a decision because ADR-007 still describes JWT RS256 as the intended design.

---

## Context

ADR-007 chose Kong and listed the `jwt` plugin as the authentication mechanism, with tokens issued by identity. The implementation never configured the plugin or any consumer. Instead:

- `01-ms-identity` issues an **opaque session id** (UUID) at login and owns the session record (`identity.user_session`, status `Active`/`Closed`).
- Kong's global `pre-function` (Lua) requires `Authorization: Bearer <uuid>` and calls `GET ms-identity:8081/api/v1/sessions/{id}` on every non-public route.
- Every service (Java `AuthTokenFilter`, TS `registerAuthGuard`, Python `require_permission`) re-checks the same session and then calls `02-ms-authorization` (`GET /auth/evaluate`) for RBAC.
- Positive checks are cached 10 s (`kong.cache`, and in each guard); logout invalidates the edge entry.

The tech debts `AT-004`/`AT-007` in `05-architecture/overview.md` describe this as a gap against the JWT target, but no ADR captures why the session model exists or that it is the current contract.

**Known constraints:**
- Immediate revocation is required for logout and for disabled users (a stolen biometric-kiosk session must be killable).
- Three runtimes (Java, Node, Python) must enforce the same rule with no shared library.
- A 3-person team; no key-rotation tooling for RS256/JWKS.

---

## Decision

**We decided:** keep **opaque server-side sessions** validated by introspection against `01-ms-identity` at the edge and again inside every service, with a **10 s positive cache**, and treat an RS256 JWT as a possible future optimisation, not the baseline.

**Justification:**
1. Revocation is exact (bounded by the 10 s cache), whereas a stateless JWT stays valid until expiry.
2. Session state (`user_session`) already exists in the data model (`06-data`), so no new infrastructure is needed.
3. Defence in depth: services do not trust the edge, so an internal call bypassing Kong is still authenticated.
4. No signing-key management or JWKS distribution across three runtimes.

---

## Evaluated alternatives

| Alternative | Pros | Cons | Reason for discarding |
|------------|------|------|-----------------------|
| **Opaque session + introspection (chosen)** | Instant revocation, simple, reuses `user_session` | Identity is called on cache miss; duplicated guard logic per service | — (chosen) |
| JWT RS256 via Kong `jwt` plugin (ADR-007 target) | No per-request identity call; stateless | Revocation needs a blocklist; key rotation; claims (roles) go stale; plugin consumers config | Not needed at current load; deferred (`AT-007`, v2.0) |
| JWT validated in each service (no Kong plugin) | Stateless | Same key distribution problem in 3 runtimes | Same as above, more code |
| Kong `key-auth` / external IdP (Keycloak) | Mature | New component, operational weight | Over-scope for one institution |

---

## Consequences

**Positive:**
- Logout and account disable take effect within 10 s everywhere.
- One contract (`GET /sessions/{id}` + `GET /auth/evaluate`) for all services.

**Negative / Trade-offs:**
- Identity is on the critical path: if it does not answer, Kong returns 503 for every protected route.
- The check is duplicated in nine services (no shared module, by design).
- The bearer is not self-describing: clients cannot read roles from it.
- Rate limits and CORS are independent of this choice (see ADR-007).

**Impact on the system:**
- Affected services: 01 (owner), 02, and all services' guards; Kong `pre-function`.
- Documents to keep consistent: `07-api/authentication.md`, `05-architecture/overview.md` (AT-004, AT-007), `00-governance/security-policy.md`, ADR-007 (target state wording).

---

## Risks

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|-----------|
| Identity outage blocks all traffic | Medium | High | Health checks, restart policy; consider longer positive cache or JWT in v2.0 |
| Guard implementations drift between runtimes | Medium | High | Keep the shared spec in `07-api/authentication.md` and test each guard against it |
| 10 s revocation window is too long for a kill-switch | Low | Medium | Cache invalidation on logout already exists; shorten TTL if needed |

---

## References

- `FULL/back-end/99-api-gateway/kong/kong.yml`, `01-ms-identity` (`AuthController`, `SessionController`, `AuthTokenFilter`), `03/07/08/09` `authGuard.ts`, `06-ms-biometric/infrastructure/web/security.py`
- Related to: ADR-007 (gateway; its JWT target is deferred by this ADR), ADR-008 (password hashing), ADR-002
