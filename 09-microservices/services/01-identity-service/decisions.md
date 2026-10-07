# Design Decisions — Identity Service

> Technical decisions specific to this service (not global architectural ones — those go in `05-architecture/decisions/`).
> Document here things that are not obvious: why this library, why this pattern, why this structure.

---

## DEC-001 Local ownership and bounded context boundaries

**Date:** 2026-09-07
**Status:** Active

**Context:**
The identity context owns the canonical data for schools, people, users, and login sessions. A local service boundary is required so the service can evolve without creating hidden coupling with academic or attendance logic.

**Decision:**
The service keeps authoritative records for `School`, `Person`, `User`, and `User_Session` and treats neighboring domains as external consumers of public IDs and events.

**Alternatives considered:**
- Shared database ownership with academic or authorization services: rejected because it erodes service boundaries and hides domain ownership.

**Consequences:**
- ✅ Clear accountability for identity records and session lifecycle
- ⚠️ Cross-context consistency depends on explicit API/event contracts

---

## DEC-002 Public-ID integration instead of cross-context foreign keys

**Date:** 2026-09-07
**Status:** Active

**Context:**
Academic, authorization, and biometric contexts need to reference identity data, but direct FK joins would create hidden dependencies and break context autonomy.

**Decision:**
Identity exposes stable public IDs such as `school_id`, `person_id`, and `user_id`, and downstream services resolve the relationship via API calls or domain events instead of DB-level foreign keys.

**Alternatives considered:**
- Direct database joins across contexts: rejected because it creates tight coupling and makes independent deployment risky.

**Consequences:**
- ✅ Better modularity and safer service evolution
- ⚠️ Requires discipline in event payloads and integration contracts

---
