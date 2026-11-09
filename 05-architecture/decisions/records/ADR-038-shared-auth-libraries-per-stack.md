# ADR-038 — Shared authentication libraries per stack — amends ADR-023

- **ID:** ADR-038
- **Date:** 2026-11-08
- **Status:** Accepted
- **Authors:** FaceAttend EDU Team
- **Related:** ADR-005 (stack per service), ADR-007 (API gateway, JWT target), ADR-023 (opaque session), ADR-033 (school scope); `05-architecture/overview.md` §8 AT-004 and AT-007; `FULL` PR #395.

---

## Context

ADR-023 states, as a constraint, that "three runtimes (Java, Node, Python) must enforce the same rule with no shared library". Each service therefore carried its own copy of the session check and of the rule that confines a caller to their school. The copies drifted: `callerSchool.ts` in academic and configuration differed by 43 lines, `CallerSchoolResolver` in scheduling and attendance used different timeouts and only one retried, and the literal `"Super Administrador"` appeared in 89 places outside tests (architecture audit, finding M5; tech debt AT-004).

The session check was already moved to `back-end/shared/auth-java` (`SessionAuthClient`) and `back-end/shared/auth-node` (`registerAuthGuard`) without a recorded decision. This ADR records that decision and its next step.

---

## Decision

**We decided:** each stack has one shared library for the parts of authentication that must behave the same in every service; each service keeps what is its own.

1. **Shared (one copy per stack):**
   - session introspection against identity and the permission check against authorization, with the positive-only 10 s cache (`SessionAuthClient`, `registerAuthGuard`);
   - the caller's school: Super Administrador is unrestricted, everyone else gets the school of their own academic actor or none, cached 30 s (`CallerSchoolLookup` in Java, `createCallerSchoolResolver` in Node);
   - the role names (`Roles`, `ROLES`).
2. **Per service:** the permission each route requires (`requiredPermission`, `PermissionRule`), how the service reads a person's actors (academic reads its own table, the others call academic), timeouts and retries, and the shape of the scope object its use cases receive.
3. **Distribution:** source in the monorepo, no registry. Java services build `shared/auth-java` first (`mvn install`, also in CI and in the Dockerfiles, whose build context is `back-end/`); Node services depend on it with `file:../shared/auth-node`. The version stays `1.0.0`: a change reaches every service of the stack in the same commit.
4. **Changes are additive.** A change to a shared library ships with its own tests and with every consuming service's suite green in the same pull request.
5. **Biometric (Python) is out of scope:** it is the only Python service, so `require_permission` stays in it until a second Python service exists.

**Justification:** these rules are security rules, and a divergence between copies is a vulnerability (a scope that one service enforces and another forgets), not a style difference. Keeping the permission rules per service preserves each context's ownership of its own authorization.

---

## Evaluated alternatives

| Alternative | Pros | Cons | Reason for discarding |
|---|---|---|---|
| Shared source library per stack, monorepo, single version — chosen | One fix reaches every service; no infrastructure to run | Services of a stack build and release together; a breaking change touches all of them | — (chosen) |
| Keep one copy per service (ADR-023 constraint) | Services fully independent | The copies already drifted; a fix must be repeated by hand | Drift is the problem this ADR solves |
| Versioned packages in a registry (GitHub Packages / npm) | Each service upgrades when ready | Registry, credentials and release process for 2 small libraries; old versions keep old bugs alive | Cost out of proportion for one team and one repository |
| Authorizer at the edge (Kong plugin or OPA) | One implementation for every runtime | Services lose defense in depth; school scope needs academic data at the edge | Kept as the long-term direction with the JWT of ADR-007 |

---

## Consequences

**Positive:**
- Super Administrador, the school lookup and the session check behave the same in every Java and Node service.
- The services' own code shrinks to their permission rules and adapters (scheduling and attendance `CallerSchoolResolver` are thin subclasses; academic and configuration `callerSchool.ts` only supply the actor lookup).

**Negative / Trade-offs:**
- Coupling between microservices of the same stack: they cannot move to different library versions.
- Builds depend on a step outside the service folder (`mvn install` of `shared/auth-java`, build context `back-end/`).
- Installing the library locally overwrites `faceattend-auth-1.0.0` in `~/.m2`, which is shared by every checkout and worktree on the machine.
- Node services link the library through `node_modules`; a worktree that reuses another checkout's `node_modules` tests the other checkout's copy.

**Impact on the system:** `back-end/shared/auth-java`, `back-end/shared/auth-node`, the `security` packages of 01/02/04/05, `callerSchool.ts` and `actorScope.ts`/`caseScope.ts` of 03/07/08, the service Dockerfiles and `.github/workflows/ci.yml`.

---

## Risks

| Risk | Probability | Impact | Mitigation |
|---|---|---|---|
| A change to a shared library breaks a service that was not tested | Medium | High | CI builds and tests every service against the library of the same commit; rule 4 |
| A stale `~/.m2` or linked `node_modules` makes local tests pass against old library code | Medium | Medium | Rebuild the library before testing; CI starts clean |
| The shared code grows into service-specific rules | Low | Medium | Rule 2: permission rules and data access stay in the services |

---

## References

- Architecture audit, finding M5; tech debt AT-004 in `05-architecture/overview.md` §8.
- `FULL` PR #395 (caller school lookup and role names shared in both stacks).
- `00-governance/security-policy.md` (service check).
- Amends the "no shared library" constraint in the context of ADR-023.
