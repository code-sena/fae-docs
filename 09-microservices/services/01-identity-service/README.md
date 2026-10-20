# Identity Service

> Owns person profiles, user credentials, login sessions, and the password policy for the platform.

---

## Responsibility

> This bounded context is responsible for identity records, authentication metadata, and session lifecycle across the institution. **It does not own `School`** — schools live in the Academic context (`03-ms-academic`).

---

## Bounded context ownership

| Entity | Description |
|--------|-------------|
| Person | Represents the natural person; school-agnostic (no `school_id`). |
| App_User | Stores credential and authentication metadata for a individual. |
| User_Session | Tracks active and closed authentication sessions. |
| Password_Policy | Password rules (length, complexity, expiration). |

> **Removed:** `City` — dropped by Liquibase (`008-drop-city-table.yaml`); cities come from the external `countriesnow.space` API.

---

## Location in the architecture

| Field | Value |
|-------|-------|
| Number in catalog | 01 |
| Local port | 8081 |
| Repository | `back-end/01-ms-identity` |
| DB engine | PostgreSQL (schema `identity`) |
| Communicates with | authorization-service (RBAC check) |
| Consumed by | API Gateway, authentication flows |

---

## Local rules

- `Person` is **not** scoped by school; uniqueness is the pair `(document_type, document_number)` plus a partial unique on `email`.
- `App_User` belongs to a single `Person` (1:1, `person_id` UNIQUE) and owns the login credential set.
- `User_Session` is a child of `App_User` and records device and session lifecycle; sessions are opaque UUIDs (no JWT, no Redis session store).
- Cross-context references are handled through public IDs rather than direct FKs.

---

## Related documents

- [data-model.md](./data-model.md) — canonical schema for the identity context
- [events.md](./events.md) — events published and consumed by this service
- [decisions.md](./decisions.md) — design decisions specific to the bounded context
- [runbook.md](./runbook.md) — operational guidance in production
- [API Contract](../../../07-api/contracts/openapi/01-identity-service.yaml)

