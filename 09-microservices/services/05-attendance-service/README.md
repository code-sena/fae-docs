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
| Number in catalog | 04 |
| Local port | 8085 |
| Repository | [Service repo URL] |
| DB engine | PostgreSQL |
| Communicates with | biometric-service, academic-service |
| Consumed by | API Gateway, staff and administrators |

---

## Local rules

- `Class_Session` represents a real class occurrence, not the generic schedule.
- Attendance is only valid while the associated session is `Open`.
- `Attendance.student_id` and `Class_Session.opened_by` are external IDs without FK to other bounded contexts.
- `Justification` is a child record of `Attendance` and keeps review and approval state.
- `schedule_id` stays as an external reference to the academic schedule.

---

## Related documents

- [data-model.md](./data-model.md) — attendance schema, statuses, and constraints
- [events.md](./events.md) — event contracts for class start, attendance, and justification updates
- [decisions.md](./decisions.md) — service decisions that shape attendance operations
- [runbook.md](./runbook.md) — incident handling and operational playbooks
- [API Contract](../../../07-api/contracts/openapi/05-attendance-service.yaml)

