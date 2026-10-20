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
| Stack | Node 20 + Fastify + Drizzle ORM (`pg`) + kafkajs + nodemailer + pino |
| Folder in repo | `FULL/back-end/08-ms-notification/` |
| Communicates with | **identity** (Identity calls its internal `POST /api/v1/emails` for password-reset mails) and **attendance** (consumes the Kafka topic `attendance.justification.events` when `KAFKA_BROKERS` is set, which compose does); it also calls an optional external SMTP/push/SMS webhook. It publishes no events of its own and has no link with biometric |
| Consumed by | Web/Mobile through Kong `:8000` (`/api/v1/alerts`, `/alert-types`) |

---

## Related documents

- [API Contract](../../../07-api/contracts/openapi/08-notification-service.yaml)
- Data model → [`06-data/domains/07-notification.md`](../../../06-data/domains/07-notification.md)
- Catalog entry → [`service-catalog.md`](../../service-catalog.md)
- Per-service pages: [`data-model.md`](data-model.md) · [`decisions.md`](decisions.md) ·
  [`events.md`](events.md) · [`runbook.md`](runbook.md)
