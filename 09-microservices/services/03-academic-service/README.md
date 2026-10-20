# Academic Service

> Owns the academic structure of the school: schools, programs, periods, cohorts, courses, academic actors, and enrollments.

---

## Responsibility

> This bounded context manages the canonical academic domain and the operational rules that keep curricular and enrollment data consistent.

---

## Bounded context ownership

| Entity | Description |
|--------|-------------|
| School | The educational institution and its master profile. |
| Program | Curricular container ("Tecnología en Análisis de Datos"). |
| Academic_Period | Academic cycle or term (`starts_on` < `ends_on`). |
| Cohort | Group advancing together ("Ficha 2691234 — ADSI — T1 2026"). |
| Course | Subject within a program ("Bases de Datos I"). |
| Academic_Actor_Type | Catalog: STUDENT / INSTRUCTOR. |
| Academic_Actor | A person acting as student or instructor in a school. |
| Enrollment | Binds an academic actor to a cohort. |

> **Note:** rooms and weekly timetables (`Environment`, `Schedule_Block`, `Class_Session`) belong to the **Scheduling** context (`04-ms-scheduling`), not here.

---

## Location in the architecture

| Field | Value |
|-------|-------|
| Number in catalog | 03 |
| Local port | 8083 |
| Repository | `back-end/03-ms-academic` |
| DB engine | PostgreSQL (schema `academic`) |
| Communicates with | none synchronously (Kong + identity→notification email are the only sync paths) |
| Consumed by | API Gateway, Scheduling, Attendance, Configuration, admin and teacher clients |

---

## Local rules

- `School` is the root aggregate for the academic context; `Program`, `Academic_Period`, and `Course` are school-scoped catalogs.
- `Academic_Actor` links a `Person` (public UUID, no cross-context FK) to a school with a type (STUDENT/INSTRUCTOR).
- `Enrollment` uniquely binds an academic actor to a cohort.
- `Schedule_Block` uniqueness (room and instructor time slots) is enforced in the Scheduling context.
- `Cohort`, `Program`, and `Course` are local school-scoped catalogs.

---

## Related documents

- [data-model.md](./data-model.md) — academic entity schema and constraints
- [events.md](./events.md) — academic events and downstream consumers
- [decisions.md](./decisions.md) — service-specific architectural decisions
- [runbook.md](./runbook.md) — operational procedures for academic service support
- [API Contract](../../../07-api/contracts/openapi/03-academic-service.yaml)

