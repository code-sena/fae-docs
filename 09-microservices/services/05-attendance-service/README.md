# Attendance Service

> Owns class sessions, daily attendance, and justification workflows for academic operations.

---

## Responsibility

> This bounded context captures when a class was opened, who attended, and how exceptions are validated and recorded.

---

## Bounded context ownership

| Entity | Description |
|--------|-------------|
| Class_Session | A real instance of a scheduled class on a specific date. |
| Attendance | Per-student attendance record for an open class session. |
| Justification_Type | Catalog of allowed absence or lateness reasons. |
| Justification | Student justification attached to an attendance record. |

---

## Location in the architecture

| Field | Value |
|-------|-------|
| Number in catalog | 05 |
| Local port | 8085 |
| Repository | `back-end/05-ms-attendance` |
| DB engine | PostgreSQL (schema `attendance`) |
| Communicates with | none synchronously (biometric and scheduling feed it via API/events, not the other way) |
| Consumed by | API Gateway, staff and administrators |

---

## Local rules

- `Class_Session` is **owned by Scheduling** (`04-ms-scheduling`); Attendance references it by public ID.
- Attendance is only valid while the associated session is `Open`.
- `Attendance_Record.academic_actor_id` and `Class_Session.opened_by` are external IDs without FK to other bounded contexts.
- `Justification` is a 1:1 child of `Attendance_Record` (UNIQUE `attendance_record_id`) and keeps review and approval state.
- `class_session_id` is the external reference to the scheduled session (there is no `schedule_id`).

---

## Related documents

- [data-model.md](./data-model.md) — attendance schema, statuses, and constraints
- [events.md](./events.md) — event contracts for class start, attendance, and justification updates
- [decisions.md](./decisions.md) — service decisions that shape attendance operations
- [runbook.md](./runbook.md) — incident handling and operational playbooks
- [API Contract](../../../07-api/contracts/openapi/05-attendance-service.yaml)

