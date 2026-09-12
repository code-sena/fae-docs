# Design Decisions — Authorization Service

> Technical decisions specific to this service (not global architectural ones — those go in `05-architecture/decisions/`).
> Document here things that are not obvious: why this library, why this pattern, why this structure.

---

## DEC-001 Role ownership is school-scoped

**Date:** 2026-09-07
**Status:** Active

**Context:**
Authorization must reflect institutional policy while remaining independent from identity data. Roles and permissions need a local source of truth that is scoped to a school but still supports global permission names.

**Decision:**
`Role` is owned within the authorization context and is keyed by `school_id` plus role name, while `Permission` remains global by permission name and is assigned to roles through `Role_Permission`.

**Alternatives considered:**
- Storing roles directly inside the identity service: rejected because it mixes identity responsibility with access policy.

**Consequences:**
- ✅ Clear policy ownership and easier school-level customization
- ⚠️ Requires role synchronization rules when a school is created or a policy template is initialized

---

## DEC-002 User-role assignments use public identity identifiers

**Date:** 2026-09-07
**Status:** Active

**Context:**
Authorization needs to assign rights to users without owning identity records or enforcing identity database constraints.

**Decision:**
`User_Role.user_id` stores the public `Identity.User.user_id` and is treated as an external identifier, not a foreign key to the identity database.

**Alternatives considered:**
- Embedding full user objects or direct identity joins: rejected because it would create a cross-context dependency and break autonomy.

**Consequences:**
- ✅ Authorization stays decoupled from identity persistence
- ⚠️ Authorization must rely on consistent event contracts and user ID stability

---
