# Design Decisions — Configuration Service

> Technical decisions specific to this service (not global architectural ones — those go in `05-architecture/decisions/`).
> Document here things that are not obvious: why this library, why this pattern, why this structure.

---

## DEC-001 School-scoped settings are key-value records within the configuration context

**Date:** 2026-09-07
**Status:** Active

**Context:**
Operations require configuration values that vary by school, such as attendance tolerances, academic limits, or other school-specific tuning. These settings should not be spread across other domains.

**Decision:**
`Academic_Configuration` is owned by the configuration service and keeps the canonical value for each `(school_id, configuration_name)` pair.

**Alternatives considered:**
- Storing configuration inside the academic service: rejected because it mixes domain data with operational tuning and reduces reuse across the platform.

**Consequences:**
- ✅ Uniform access to operational settings across services
- ⚠️ Requires a clear configuration update propagation model

---

## DEC-002 Security policy is intentionally separated from business entities

**Date:** 2026-09-07
**Status:** Active

**Context:**
Password policy and security settings are platform concerns and should not be coupled to users, schools, or academic records in a way that creates hidden dependency chains.

**Decision:**
`Security_Configuration` and `Password_Policy` are treated as platform-level configuration records owned by the configuration context and used as operational defaults by the security and identity flows.

**Alternatives considered:**
- Embedding policy rules into identity or application code: rejected because it makes security controls harder to govern and update centrally.

**Consequences:**
- ✅ Centralized governance for policy changes and rollout control
- ⚠️ Requires explicit propagation to any service consuming policy data

---
