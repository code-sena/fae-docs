# Design Decisions — Audit Service

> Technical decisions specific to this service (not global architectural ones — those go in `05-architecture/decisions/`).
> Document here things that are not obvious: why this library, why this pattern, why this structure.

---

## DEC-001 Audit records are append-only and context-agnostic

**Date:** 2026-09-07
**Status:** Active

**Context:**
Audit data must support forensic review and operational traceability without depending on the state of a business owner service. The records should remain stable even if the original domain data changes.

**Decision:**
The audit context stores immutable or append-friendly records in its own schema, keyed by event metadata and aggregate identifiers rather than by live foreign keys to transactional services.

**Alternatives considered:**
- Storing audit trails in business service databases: rejected because it couples operational evidence to the lifecycle of the source service and makes auditing brittle.

**Consequences:**
- ✅ Strong traceability and better compliance support
- ⚠️ Requires consumers to publish structured events consistently

---

## DEC-002 External actor IDs are intentionally not enforced as foreign keys

**Date:** 2026-09-07
**Status:** Active

**Context:**
An audit event can reference a user or system actor that may already be deleted, anonymized, or outside the current identity store, yet the event must remain meaningful.

**Decision:**
`actor_id`, `user_id`, and similar fields are stored as plain IDs to preserve history, and the audit service avoids cross-context FK constraints.

**Alternatives considered:**
- Direct references to active identity or user records: rejected because it breaks historical traceability and would fail under deleted-user scenarios.

**Consequences:**
- ✅ More resilient historical records and stronger forensic value
- ⚠️ Some relationship context is intentionally not database-enforced

---
