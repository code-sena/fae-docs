# IoT Service

> Owns physical devices, classroom assignments, and operational state of hardware used by the attendance platform.

---

## Responsibility

> This bounded context keeps device identity and live assignment state independent from the academic and attendance domain data.

---

## Bounded context ownership

| Entity | Description |
|--------|-------------|
| IoT_Device | Registered device in the school environment. |
| Device_Assignment | Historical and active assignment of a device to a classroom. |

---

## Location in the architecture

| Field | Value |
|-------|-------|
| Number in catalog | 06 |
| Local port | 8087 |
| Repository | [Service repo URL] |
| DB engine | PostgreSQL |
| Communicates with | attendance-service, biometric-service, configuration-service |
| Consumed by | Classroom devices, gateways, operations team |

---

## Local rules

- `IoT_Device` is school-scoped and tracks physical device metadata and status.
- `Device_Assignment` stores the lifecycle of a device in a classroom over time.
- `classroom_id` is kept as a public reference instead of a cross-context FK.
- A device may be reassigned to another classroom while preserving historical assignment records.

---

## Related documents

- [data-model.md](./data-model.md) — device and assignment schema
- [events.md](./events.md) — device lifecycle and readiness events
- [decisions.md](./decisions.md) — operational and integration decisions for IoT assets
- [runbook.md](./runbook.md) — incident response for devices and connectivity
- [API Contract](../../../07-api/contracts/openapi/06-iot-service.yaml)

