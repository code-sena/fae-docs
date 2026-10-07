# Design Decisions — IoT Service

> Technical decisions specific to this service (not global architectural ones — those go in `05-architecture/decisions/`).
> Document here things that are not obvious: why this library, why this pattern, why this structure.

---

## DEC-001 Device lifecycle is independent from classroom ownership

**Date:** 2026-09-07
**Status:** Active

**Context:**
IoT hardware can be reassigned, repaired, or moved between rooms without changing the academic domain model. Classroom assignment should not be treated as a hard DB dependency.

**Decision:**
The IoT service owns device identity and assignment history, while `classroom_id` remains a public reference from the academic context rather than a direct FK.

**Alternatives considered:**
- Keeping classroom assignment as a direct relational dependency in the academic schema: rejected because it couples hardware lifecycle to course and room management.

**Consequences:**
- ✅ Device mobility and maintenance history stay local to IoT
- ⚠️ Assignment changes require event publication for attendance and monitoring consumers

---

## DEC-002 Historical assignment tracking is part of the device domain

**Date:** 2026-09-07
**Status:** Active

**Context:**
The platform needs to know where a device was assigned over time, not only where it is currently located.

**Decision:**
`Device_Assignment` stores interval-based history with `assigned_from`, `assigned_to`, and status, allowing the service to preserve the operational timeline without being tied to the academic lifecycle rules.

**Alternatives considered:**
- Oversimplifying to a single current classroom assignment: rejected because it discards operational history and auditability.

**Consequences:**
- ✅ Better operational traceability and maintenance support
- ⚠️ Requires integrity rules around overlapping assignments and reassignment windows

---
