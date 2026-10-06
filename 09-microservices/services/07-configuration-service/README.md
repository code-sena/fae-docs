# Configuration Service

> Owns platform and academic configuration values that control behavior without being part of the core business entities.

---

## Responsibility

> This bounded context centralizes settings that must be shared or tuned operationally across services, including academic and security policies.

---

## Bounded context ownership

| Entity | Description |
|--------|-------------|
| Academic_Configuration | Configurable academic parameter by school. |
| Security_Configuration | Global security configuration values. |
| Password_Policy | Password complexity and expiration rules. |

---

## Location in the architecture

| Field | Value |
|-------|-------|
| Number in catalog | 08 |
| Local port | 8087 |
| Repository | [Service repo URL] |
| DB engine | PostgreSQL |
| Communicates with | academic-service, attendance-service, identity-service |
| Consumed by | Operations, administrators, platform services |

---

## Local rules

- `Academic_Configuration` is school-scoped and key/value-based.
- `Security_Configuration` is a shared policy layer not tied to a specific user.
- `Password_Policy` is a system-wide default for credentials and lifecycle enforcement.
- Configuration values must be treated as operational metadata, not as source-of-truth business records.

---

## Related documents

- [data-model.md](./data-model.md) — configuration schema and settings model
- [events.md](./events.md) — change events for configuration propagation
- [decisions.md](./decisions.md) — service governance and policy decisions
- [runbook.md](./runbook.md) — configuration rollout and rollback procedures
- [API Contract](../../../07-api/contracts/openapi/07-configuration-service.yaml)

