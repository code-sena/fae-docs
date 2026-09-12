# Identity Service

> Owns school master data, person profiles, credentials, and login sessions for the platform.

---

## Responsibility

> This bounded context is responsible for identity records, authentication metadata, and session lifecycle across the institution.

---

## Bounded context ownership

| Entity | Description |
|--------|-------------|
| School | Defines the educational institution and its master profile. |
| Person | Represents the natural person associated with the school. |
| User | Stores credential and authentication metadata for an individual. |
| User_Session | Tracks active and closed authentication sessions. |

---

## Location in the architecture

| Field | Value |
|-------|-------|
| Number in catalog | 01 |
| Local port | 8081 |
| Repository | [Service repo URL] |
| DB engine | PostgreSQL |
| Communicates with | authorization-service, audit-service |
| Consumed by | API Gateway, authentication flows |

---

## Local rules

- `School` is the root aggregate for institutional identity.
- `Person` is unique per school and keeps profile and contact details.
- `User` belongs to a single `Person` and owns the login credential set.
- `User_Session` is a child of `User` and records device and session lifecycle.
- Cross-context references are handled through public IDs rather than direct FKs.

---

## Related documents

- [data-model.md](./data-model.md) — canonical schema for the identity context
- [events.md](./events.md) — events published and consumed by this service
- [decisions.md](./decisions.md) — design decisions specific to the bounded context
- [runbook.md](./runbook.md) — operational guidance in production
- [API Contract](../../../07-api/contracts/openapi/01-identity-service.yaml)

