# ADR-036 — Pin third-party images by digest, tag own images by commit, fix advisories with overrides

- **ID:** ADR-036
- **Date:** 2026-10-28
- **Status:** Accepted
- **Authors:** FaceAttend EDU Team
- **Related:** ADR-018 (CI), ADR-020 (deployment), ADR-031 (release gate); `FULL` PR #300, #302, #301 and #312.

---

## Context

The compose files used floating tags (`mongo:7`, `redis:7-alpine`, `kong:3.6`, `postgres:17`) and our own images used `:latest` (finding OPS-03). The same `docker compose up` could start different code on two days, and a rollback had no stable target. Dependency audits also reported high advisories in transitive packages that the direct dependency had not yet updated (finding S-NEW-05).

---

## Decision

**We decided:**
1. **Third-party images** are pinned as `tag@sha256:digest` (mongo, redis, kong, kafka, postgres, liquibase, caddy, portainer). The tag stays for readability; the digest decides. Digests are reviewed by hand while Dependabot is paused (procedure in `FULL/COMPOSE.md`).
2. **Own images** take `${FACEATTEND_IMAGE_TAG:-latest}`; deployments use the commit SHA, so a rollback is changing one variable. `latest` is for local work only.
3. **Transitive advisories** are fixed with a package-manager override (`overrides` in npm, a lock update in Poetry) when a patched version exists and the build and tests pass with it; when no patched version exists, the advisory is recorded and accepted with its reason.

**Justification:** reproducible builds and rollbacks are a precondition for the release gate (ADR-031); overrides fix the risk without waiting for, or downgrading, the direct dependency.

---

## Evaluated alternatives

| Alternative | Pros | Cons | Reason for discarding |
|---|---|---|---|
| Digest pinning + SHA tags + overrides — chosen | Reproducible, cheap rollbacks | Manual digest updates | — (chosen) |
| Exact version tags only (`mongo:7.0.43`) | Readable | A tag can be re-pushed | Digest is the only immutable reference |
| Keep floating tags | Automatic patches | Non-reproducible, surprise upgrades | OPS-03 |
| Downgrade Expo to remove the zod advisory (`npm audit fix --force`) | Clean audit | Breaks the app (Expo 51) | Out of proportion |

---

## Consequences

**Positive:** the stack started from a commit is the same every time; `npm audit --omit=dev` shows no high advisories in Web and Mobile; `pip-audit` shows none in ms-biometric.

**Negative / Trade-offs:**
- Security patches of base images are not picked up automatically: someone must refresh digests.
- `zod` is overridden from 3 to 4 for `@expo/cli`; exercised with `expo export` and `expo config`, but an EAS build must be tried before the next mobile release.
- Base images inside the Dockerfiles (`node`, `eclipse-temurin`, `maven`, `python`, `nginx`) are still floating.
- 8 low `deepmerge` advisories remain in Web and Mobile: no patched version exists.

**Impact on the system:** `docker-compose*.yml`, `.env.example`, Web and Mobile `package.json`, ms-biometric `poetry.lock`.

---

## Risks

| Risk | Probability | Impact | Mitigation |
|---|---|---|---|
| Digests age and miss security fixes | Medium | Medium | Monthly review of digests; re-enable Dependabot for docker |
| An override breaks a tool at build time | Low | Medium | Overrides only with tests and a build passing |

---

## References

- Audit findings OPS-03 and S-NEW-05; `FULL` PR #300 (digests, verified on the local stack), #302 (image tag variable), #301 (dnspython 2.9.0), #312 (zod 4.6.7).
- Related to: ADR-018, ADR-020, ADR-031.
