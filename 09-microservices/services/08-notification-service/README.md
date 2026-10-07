# Notification Service

> Owns alert types and the alert lifecycle (absenteeism, tardiness, low attendance, pending
> justifications, biometric updates) raised for academic actors.

---

## Responsibility

> This bounded context stores `alert_type` (catalog) and `alert` (raised → resolved) rows that
> the Web/Mobile dashboards read. Alerts reference `academic_actor_id` as a public ID — no FK.

---

## Bounded context ownership

| Entity | Description |
|--------|-------------|
| Alert_Type | Catalog of alert codes with severity and channel |
| Alert | Instance of an alert for an academic actor, with raised/resolved timestamps |

---

## Location in the architecture

| Field | Value |
|-------|-------|
| Number in catalog | 08 |
| Local port | 8088 |
| Schema | `notification` (PostgreSQL 17) |
| Stack | Go 1.22 + Gin + pgx + zap |
| Folder in repo | `FULL/back-end/08-ms-notification/` |
| Communicates with | attendance, biometric (event-driven via Kafka `:9092`) |
| Consumed by | Web/Mobile through Kong `:8080` (`/api/v1/alerts`, `/alert-types`) |

---

## Related documents

- [API Contract](../../../07-api/contracts/openapi/08-notification-service.yaml)
- Data model → [`06-data/domains/07-notification.md`](../../../06-data/domains/07-notification.md)
- Catalog entry → [`service-catalog.md`](../../service-catalog.md)
- Per-service pages (`data-model.md`, `events.md`, `decisions.md`, `runbook.md`) are **pending** —
  follow `../../_template/service/` when writing them.
