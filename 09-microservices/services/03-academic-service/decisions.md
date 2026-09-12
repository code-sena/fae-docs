# Design Decisions — Academic Service

> Technical decisions specific to this service (not global architectural ones — those go in `05-architecture/decisions/`).
> Document here things that are not obvious: why this library, why this pattern, why this structure.

---

## DEC-001 Academic profiles are separate from the identity person

**Date:** 2026-09-07
**Status:** Active

**Context:**
The school has a person profile in Identity, but academic rules such as student and teacher enrollment require a separate academic representation that can evolve independently.

**Decision:**
`Student` and `Teacher` are themselves first-class academic aggregates owned by the academic context, each linked through a public `person_id` to the identity record without a direct FK.

**Alternatives considered:**
- Reusing `Person` as the academic aggregate: rejected because it would collapse two different domains into one entity and weaken boundary clarity.

**Consequences:**
- ✅ Academic lifecycle is independent of identity lifecycle
- ⚠️ Requires explicit mapping between person and academic profile during creation flows

---

## DEC-002 Enrollment and schedule uniqueness rules are local invariants

**Date:** 2026-09-07
**Status:** Active

**Context:**
Academic integrity depends on ensuring the same student is not enrolled in duplicate seat assignments, and teacher/classroom schedules do not collide for a given period.

**Decision:**
The academic service enforces local uniqueness constraints such as `(student_id, course_id, period_id)` for enrollment and classroom/teacher slot checks in `Schedule` within its own database.

**Alternatives considered:**
- Enforcing these rules centrally in the attendance or identity services: rejected because the business invariants belong to the academic context itself.

**Consequences:**
- ✅ Strong local consistency for curricular operations
- ⚠️ The service must publish these changes as events for downstream consumers

---
