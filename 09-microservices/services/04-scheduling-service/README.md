# Scheduling Service

> Owns the weekly teaching grid (environments, schedule blocks) and its materialization into
> concrete class sessions, enforcing the anti-double-booking rules of the scheduling context.

---

## Responsibility

> This bounded context turns "who teaches what, where and when" into `class_session` rows that
> attendance consumes. It never touches enrollment or person data — those are public IDs.

---

## Bounded context ownership

| Entity | Description |
|--------|-------------|
| Environment | Classroom / lab of a school, with capacity |
| Schedule_Block | Recurring weekly slot (day_of_week 1-7 ISO + start/end time) |
| Class_Session | Materialization of a block on a concrete `session_date` (Open/Closed/Cancelled) |

---

## Location in the architecture

| Field | Value |
|-------|-------|
| Number in catalog | 04 |
| Local port | 8084 |
| Schema | `scheduling` (PostgreSQL 17) |
| Stack | Java 21 + Spring Boot 4.1, Maven |
| Folder in repo | `FULL/back-end/04-ms-scheduling/` |
| Communicates with | academic (public IDs), attendance (`class_session_id`) |
| Consumed by | Web/Mobile through Kong `:8080` (`/api/v1/environments`, `/schedule-blocks`, `/class-sessions`) |

---

## Related documents

- [API Contract](../../../07-api/contracts/openapi/04-scheduling-service.yaml)
- Data model → [`06-data/domains/04-scheduling.md`](../../../06-data/domains/04-scheduling.md)
- Catalog entry → [`service-catalog.md`](../../service-catalog.md)
- Per-service pages (`data-model.md`, `events.md`, `decisions.md`, `runbook.md`) are **pending** —
  follow `../../_template/service/` when writing them.
