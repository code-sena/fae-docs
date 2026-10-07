# IoT Service (future — not implemented)

> **Status (2026-10-05):** not part of the deployed system. There is **no `0X-ms-iot` service,
> no container, no port and no OpenAPI contract**; the folder is kept as design material for the
> future classroom-hardware integration (`01-context/scope.md`, `02-domain/domain-map.md`).
> The directory prefix `_future-` keeps it out of the numbered service folders, whose numbers
> match the real service numbers in `FULL/docker-compose.yml`.
>
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
| Number in catalog | — (not in the catalog: future supporting context) |
| Local port | — (not implemented) |
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
- API Contract: none yet (planned when the hardware integration is scheduled)

