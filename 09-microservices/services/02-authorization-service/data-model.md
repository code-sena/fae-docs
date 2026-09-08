# Data Model — Authorization Service

> This service is the authoritative owner of the authorization policy model.
> It owns role and permission assignments but not the identity records themselves.

---

## Database engine

**Engine:** PostgreSQL
**Justification:** Authorization rules are relational, consistent, and highly queryable for permission validation.

---

## Schema

### Table: Role

| Field | Type | Nullable | Description | Constraints |
|-------|------|----------|-------------|-------------|
| role_id | INT | No | Role identifier | PK, auto-increment |
| school_id | UUID | No | School that owns the role | Not null |
| role_name | VARCHAR | No | Name of the role | Not null |
| description | VARCHAR | Yes | Role description | Optional |
| created_at | DATETIME | No | Creation time | Not null |

**Indexes:**
- `uq_role_name_school` on `(school_id, role_name)` unique

### Table: Permission

| Field | Type | Nullable | Description | Constraints |
|-------|------|----------|-------------|-------------|
| permission_id | INT | No | Permission identifier | PK, auto-increment |
| permission_name | VARCHAR | No | Permission code or name | Not null |
| description | VARCHAR | Yes | Meaning or scope | Optional |

**Indexes:**
- `uq_permission_name` on `(permission_name)` unique

### Table: Role_Permission

| Field | Type | Nullable | Description | Constraints |
|-------|------|----------|-------------|-------------|
| role_id | INT | No | Linked role | FK to `Role.role_id` |
| permission_id | INT | No | Linked permission | FK to `Permission.permission_id` |
| assignment_date | DATETIME | No | When the permission was granted to the role | Not null |

**Indexes:**
- Primary key on `(role_id, permission_id)`

### Table: User_Role

| Field | Type | Nullable | Description | Constraints |
|-------|------|----------|-------------|-------------|
| user_id | UUID | No | User identifier in Identity context | External reference, no FK |
| role_id | INT | No | Role assigned to the user | FK to `Role.role_id` |
| assignment_date | DATETIME | No | Assignment date | Not null |

**Indexes:**
- Primary key on `(user_id, role_id)`

---

## Relationships

- `Role_Permission.role_id` → `Role.role_id`
- `Role_Permission.permission_id` → `Permission.permission_id`
- `User_Role.role_id` → `Role.role_id`

> `user_id` is a public ID from the Identity bounded context; it is not a direct cross-context FK.

---

## Modeling decisions

- **Authorization is policy-focused:** permissions and roles are modeled independently from identity details.
- **Identity remains external:** `User` belongs to the Identity service; Authorization only references the user ID.
- **Core principle:** the permission matrix is local to the Authorization context and its own relational integrity.

---

## Schema migration

**Tool:** Flyway
**Script location:** src/migrations/
**Naming convention:** V[NNN]__[description].sql

**Rollback policy:**
- Use additive migrations for new roles and permissions.
- Avoid destructive changes to existing policy assignments.

