# ADR-032 — Store only a digest of the session token

- **ID:** ADR-032
- **Date:** 2026-10-28
- **Status:** Accepted
- **Authors:** FaceAttend EDU Team
- **Related:** ADR-023 (opaque session authentication; this ADR amends how the token is stored), `FULL` PR #296 and #298.

---

## Context

ADR-023 made the bearer an opaque session UUID that identity validates. The row key of `identity.user_session` **was the bearer itself**, so anyone who could read the table (a leaked backup, a support query, a future injection) could act as every active session. Audit finding N-04 (high).

---

## Decision

**We decided:** the client keeps the raw token; identity stores only a digest of it as `user_session.session_id`.

1. When a session is opened, identity generates the token (UUID), stores `SessionTokenHasher.hash(token)` (SHA-256 over a domain prefix `faceattend.session.v1:` plus the 16 UUID bytes, folded back into a UUID) and returns the raw token once.
2. Every lookup by token (validate, refresh, logout, change password) hashes first; administrative operations by stored id (list, close another user's session) use the digest.
3. No schema change: the digest still fits `session_id UUID`.

**Justification:** a digest of a 122-bit random token cannot be reversed or brute-forced in practice, so a read of the table no longer yields bearers. A keyed HMAC would add a secret to rotate without a real gain for random tokens.

---

## Evaluated alternatives

| Alternative | Pros | Cons | Reason for discarding |
|---|---|---|---|
| SHA-256 digest of the token — chosen | No new secret, no schema change | All sessions open at deploy time stop working | — (chosen) |
| HMAC with a server key | Also protects low-entropy tokens | One more key to manage and rotate | The token is random; the key adds cost, not safety |
| Encrypt the token (AES) | Reversible for support | Reversible means stealable with the key | Nobody needs the raw token back |
| Keep the raw token | Simplest | N-04 stays open | Unacceptable |

---

## Consequences

**Positive:** a database or backup leak does not expose usable sessions; legacy rows with the raw token are inert (looked up by digest, they are never found).

**Negative / Trade-offs:**
- Deploying it logs every user out once.
- Any code that compares a stored id with a presented token must hash first. The first version missed one place: `change-password` closed the caller's own session (found by testing on the real stack, fixed in #298 with a regression test using different stored and presented ids).

**Impact on the system:** `01-ms-identity` (persistence adapter, `CloseUserSession`, `ChangePassword`); clients unchanged.

---

## Risks

| Risk | Probability | Impact | Mitigation |
|---|---|---|---|
| Another comparison of stored id vs token is added later | Medium | Medium | Regression tests use different stored and presented ids; reviewers check `getSessionId()` uses |
| Old raw-token rows kept in the table | Low | Low | They are inert; purge them with a migration when convenient |

---

## References

- Audit finding N-04; `FULL` PR #296 (digest) and #298 (change-password fix), verified on the local stack with PostgreSQL and Kong.
- Related to: ADR-023, ADR-008.
