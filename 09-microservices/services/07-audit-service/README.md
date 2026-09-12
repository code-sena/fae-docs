# Audit Service

> Owns immutable operational records, event traces, and error logs that support forensic review and compliance.

---

## Responsibility

> This bounded context records who did what, when, and from which application context without depending on the active identity model.

---

## Bounded context ownership

| Entity | Description |
|--------|-------------|
| Audit_Event | Business and operational event audit trail. |
| Error_Log | Technical and application errors captured for monitoring. |

---

## Location in the architecture

| Field | Value |
|-------|-------|
| Number in catalog | 07 |
| Local port | 8088 |
| Repository | [Service repo URL] |
| DB engine | PostgreSQL |
| Communicates with | all services via events |
| Consumed by | Operations and compliance stakeholders |

---

## Local rules

- `actor_id` and `user_id` are external identifiers and are not enforced as FK dependencies.
- `Audit_Event` is append-only and designed for correlation across services.
- `Error_Log` supports traceability and operational diagnostics.
- This service is intentionally decoupled from transactional ownership of business entities.

---

## Related documents

- [data-model.md](./data-model.md) — audit and error schema
- [events.md](./events.md) — events captured and exposed by the audit stream
- [decisions.md](./decisions.md) — retention, immutability, and governance decisions
- [runbook.md](./runbook.md) — investigation and recovery procedures
- [API Contract](../../../07-api/contracts/openapi/07-audit-service.yaml)

