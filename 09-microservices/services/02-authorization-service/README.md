# Authorization Service

> Owns roles, permissions, and the assignments that authorize access across the platform.

---

## Responsibility

> This bounded context defines authorization policies and grants for users in the institution, without owning the identity profile itself.

---

## Bounded context ownership

| Entity | Description |
|--------|-------------|
| Role | Defines a business role within a school. |
| Permission | Represents an access capability or permission name. |
| Role_Permission | Maps a role to the permissions it grants. |
| User_Role | Maps a user to one or many roles. |

---

## Location in the architecture

| Field | Value |
|-------|-------|
| Number in catalog | 02 |
| Local port | 8082 |
| Repository | `back-end/02-ms-authorization` |
| DB engine | PostgreSQL (schema `authorization`) |
| Communicates with | identity-service (session/permission evaluation) |
| Consumed by | API Gateway and internal services |

---

## Local rules

- `Role` is **global** — it has **no `school_id`**; the same role serves every school.
- `Permission` is centralized by permission name (`contexto.recurso:acción`), independent of school.
- `Role_Permission` is the association table that grants capabilities to roles.
- `User_Role` is the link between the external `user_id` and authorization roles.
- `user_id` is not a local FK; it points to the Identity context by public ID.

---

## Related documents

- [data-model.md](./data-model.md) — schema and constraints for authorization data
- [events.md](./events.md) — event interfaces for policy and role changes
- [decisions.md](./decisions.md) — service-specific decisions and trade-offs
- [runbook.md](./runbook.md) — operational procedures for production support
- [API Contract](../../../07-api/contracts/openapi/02-authorization-service.yaml)

