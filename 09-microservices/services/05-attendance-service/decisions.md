# Design Decisions — Attendance Service

> Technical decisions specific to this service (not global architectural ones — those go in `05-architecture/decisions/`).
> Document here things that are not obvious: why this library, why this pattern, why this structure.

---

## DEC-001 Class sessions are real occurrences, not generic schedules

**Date:** 2026-09-07
**Status:** Active

**Context:**
Attendance needs to reflect real class delivery, including a specific date and the teacher opening or closing the session. Reusing the generic `Schedule` record for every attendance event would blur operational reality with the instructional plan.

**Decision:**
The service creates `Class_Session` as a concrete execution record tied to a schedule and date, and attendance is recorded only against that session.

**Alternatives considered:**
- Recording attendance directly against `Schedule` and date attributes: rejected because it loses the operational lifecycle of class opening, closure, and cancellation.

**Consequences:**
- ✅ Precise operational traceability for class participation
- ⚠️ Requires explicit session lifecycle management and event publication

---

## DEC-002 Attendance references external IDs, not other-context FKs

**Date:** 2026-09-07
**Status:** Active

**Context:**
Attendance interacts with students, teachers, schedules, devices, and classroom data owned elsewhere. Direct database links would create hidden dependencies across contexts.

**Decision:**
`Attendance.student_id`, `Class_Session.opened_by`, `Class_Session.closed_by`, and the associated `schedule_id` are stored as public external identifiers without cross-context FK enforcement.

**Alternatives considered:**
- Shared database joins with academic or identity schemas: rejected because it violates bounded-context independence and makes the service less portable.

**Consequences:**
- ✅ Clean separation of attendance operational data from academic ownership
- ⚠️ Requires careful validation and event-driven synchronization for business context

---
