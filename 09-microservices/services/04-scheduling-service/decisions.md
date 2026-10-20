# Design Decisions — Scheduling Service

> Technical decisions specific to this service (not global architectural ones — those go in `05-architecture/decisions/`).
> Document here things that are not obvious: why this library, why this pattern, why this structure.

---

## DEC-001 Double-booking is prevented by unique constraints, not by application checks

**Date:** 2026-09-10
**Status:** Active (verified in DDL, 2026-10-17)

**Context:**
Two concurrent requests could create overlapping blocks for the same room or the same instructor
even if each request checks for overlaps first (classic check-then-act race).

**Decision:**
The database enforces slot uniqueness with `uq_block_environment_slot (environment_id, day_of_week, starts_at)`
and `uq_block_instructor_slot (instructor_actor_id, day_of_week, starts_at)`. The service maps the
unique violation to a client error ("slot not available") instead of silently accepting.

**Alternatives considered:**
- Advisory locks / `SELECT FOR UPDATE` on room and instructor: rejected — more moving parts for the
  same guarantee, and it only protects writers that cooperate.

**Consequences:**
- ✅ Correctness does not depend on service code, instance count, or request ordering
- ⚠️ Uniqueness is on `starts_at` only: two blocks starting at the same time are rejected even if
  their `ends_at` differ

---

## DEC-002 Sessions are materialized rows, unique per (block, date)

**Date:** 2026-09-10
**Status:** Active

**Context:**
Attendance needs a stable target per class occurrence, and an operator needs to cancel or close an
occurrence independently of the weekly plan.

**Decision:**
`class_session` is created explicitly per `(schedule_block_id, session_date)`
(`uq_session_block_date`) with lifecycle `Open → Closed | Cancelled`; closing records `closed_by`
and `closed_at`.

**Alternatives considered:**
- Deriving occurrences on the fly from `schedule_block`: rejected — no place to store open/close
  state or cancellation, and Attendance would have to recompute the plan on every query.

**Consequences:**
- ✅ Attendance writes against a real row; "who opened/closed this session" is first-class data
- ⚠️ Someone (instructor or API client) must materialize the session before attendance can be recorded

---

## DEC-003 Cross-context references are public IDs without FK

**Date:** 2026-09-10
**Status:** Active

**Context:**
`cohort_id`, `course_id`, `instructor_actor_id`, `opened_by`/`closed_by` and `environment.school_id`
point at rows owned by Academic — a different bounded context with its own schema.

**Decision:**
No cross-schema FK is declared (rule in `AGENTS.md` §2.1). The application validates the reference;
each schema can migrate independently.

**Alternatives considered:**
- Cross-schema FKs: rejected — couples deployments of two bounded contexts and breaks the
  one-schema-per-context ownership rule.

**Consequences:**
- ✅ Scheduling deploys and migrates without waiting on Academic
- ⚠️ Referential integrity is application-level: a dangling `cohort_id` is possible and must be
  handled defensively
