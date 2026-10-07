# Data Model — Configuration Service

> This service is the authoritative owner of operational configuration values used across the platform.
> It centralizes settings that affect behavior without owning the primary business records.

---

## Database engine

**Engine:** PostgreSQL
**Justification:** Configuration values are relational, versionable, and easy to validate with key uniqueness rules.

---

## Schema

### Table: Academic_Configuration

| Field | Type | Nullable | Description | Constraints |
|-------|------|----------|-------------|-------------|
| configuration_id | INT | No | Configuration identifier | PK, auto-increment |
| school_id | UUID | No | Owning school | Not null |
| configuration_name | VARCHAR | No | Configuration key | Not null |
| configuration_value | VARCHAR | No | Value of the setting | Not null |
| description | VARCHAR | Yes | Purpose of the setting | Optional |
| created_at | DATETIME | No | Creation time | Not null |
| updated_at | DATETIME | Yes | Last update | Nullable |

**Indexes:**
- `uq_academic_configuration` on `(school_id, configuration_name)` unique

### Table: Security_Configuration

| Field | Type | Nullable | Description | Constraints |
|-------|------|----------|-------------|-------------|
| configuration_id | INT | No | Security configuration identifier | PK, auto-increment |
| configuration_name | VARCHAR | No | Name of the setting | Not null |
| configuration_value | VARCHAR | No | Setting value | Not null |
| description | VARCHAR | Yes | Definition of the setting | Optional |
| created_at | DATETIME | No | Creation time | Not null |
| updated_at | DATETIME | Yes | Last update | Nullable |

### Table: Password_Policy

| Field | Type | Nullable | Description | Constraints |
|-------|------|----------|-------------|-------------|
| policy_id | INT | No | Policy identifier | PK, auto-increment |
| min_length | INT | No | Minimum password length | Default 8 |
| max_length | INT | No | Maximum password length | Default 20 |
| requires_uppercase | BOOLEAN | No | Uppercase requirement | Default true |
| requires_numbers | BOOLEAN | No | Number requirement | Default true |
| requires_symbols | BOOLEAN | No | Symbol requirement | Default true |
| expiration_days | INT | No | Password expiration in days | Default 90 |
| created_at | DATETIME | No | Creation time | Not null |
| updated_at | DATETIME | Yes | Last update | Nullable |

---

## Relationships

- None among these configuration entities beyond local settings records.
- They are intentionally decoupled from user and academic records to preserve operational independence.

---

## Modeling decisions

- **Configuration as metadata:** this service stores settings, not business facts.
- **School-scoped overrides:** `Academic_Configuration` is the only school-scoped setting in this model.
- **Shared defaults:** `Security_Configuration` and `Password_Policy` provide consistent defaults across services.

---

## Schema migration

**Tool:** Liquibase 4.29 (chained containers, ADR-010)
**Script location:** src/migrations/
**Naming convention:** V[NNN]__[description].sql

**Rollback policy:**
- Prefer backward-compatible updates to existing configuration values.
- Validate propagation endpoints before altering defaults used by other services.

