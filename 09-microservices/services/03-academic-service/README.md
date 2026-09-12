# Academic Service

> Owns the academic structure of the school: students, teachers, courses, periods, classrooms, enrollments, and schedules.

---

## Responsibility

> This bounded context manages the canonical academic domain and the operational rules that keep curricular and enrollment data consistent.

---

## Bounded context ownership

| Entity | Description |
|--------|-------------|
| Student | Academic profile linked to a person and a school. |
| Teacher | Academic profile linked to a person and a school. |
| Course | Academic subject or curricular offering. |
| Classroom | Physical learning space for a school. |
| Period | Academic cycle or term. |
| Enrollment | Student course registration per period. |
| Schedule | Timetable with teacher, course, classroom, and day/time. |

---

## Location in the architecture

| Field | Value |
|-------|-------|
| Number in catalog | 03 |
| Local port | 8084 |
| Repository | [Service repo URL] |
| DB engine | PostgreSQL |
| Communicates with | attendance-service, configuration-service, audit-service |
| Consumed by | API Gateway, admin and teacher clients |

---

## Local rules

- `Student` and `Teacher` are separate academic profiles, not direct ownership of `Person`.
- `person_id` is stored as a public ID and is not a cross-context FK.
- `Enrollment` uniquely binds a student to a course and period.
- `Schedule` enforces classroom and teacher time-slot uniqueness within a period.
- `Course`, `Classroom`, and `Period` are local school-scoped catalogs.

---

## Related documents

- [data-model.md](./data-model.md) — academic entity schema and constraints
- [events.md](./events.md) — academic events and downstream consumers
- [decisions.md](./decisions.md) — service-specific architectural decisions
- [runbook.md](./runbook.md) — operational procedures for academic service support
- [API Contract](../../../07-api/contracts/openapi/03-academic-service.yaml)

