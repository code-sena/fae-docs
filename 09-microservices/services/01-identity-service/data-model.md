# Data Model — Identity Service

> This service is the authoritative owner of the school identity data for the institution.
> No other service should read these records directly as the system of record.

---

## Database engine

**Engine:** PostgreSQL
**Justification:** Identity data is relational, transactional, and strongly constrained by uniqueness rules and login/session auditing.

---

## Schema

### Enum: Blood_Type

| Value |
|-------|
| A+ |
| A- |
| B+ |
| B- |
| AB+ |
| AB- |
| O+ |
| O- |

### Table: School

| Field | Type | Nullable | Description | Constraints |
|-------|------|----------|-------------|-------------|
| school_id | UUID | No | School identifier | PK |
| name | VARCHAR | No | School name | Not null |
| nit | VARCHAR | No | Institutional tax identifier | Unique |
| address | VARCHAR | Yes | Physical address | Optional |
| district | VARCHAR | Yes | Geographic zone or district | Optional |
| phone | VARCHAR | Yes | Main phone number | Optional |
| email | VARCHAR | Yes | Institutional email | Optional |
| status | BOOLEAN | No | School active or inactive | Default true |
| created_at | DATETIME | No | Registry creation time | Not null |
| updated_at | DATETIME | Yes | Last update time | Nullable |

**Indexes:**
- `uq_school_nit` on `(nit)` unique

### Table: Person

| Field | Type | Nullable | Description | Constraints |
|-------|------|----------|-------------|-------------|
| person_id | UUID | No | Person identifier | PK |
| school_id | UUID | No | Owner school | FK to `School.school_id` |
| name | VARCHAR | No | First name | Not null |
| last_name | VARCHAR | No | Last name | Not null |
| email | VARCHAR | Yes | Personal email | Optional |
| phone | VARCHAR | Yes | Phone number | Optional |
| rh | Blood_Type | Yes | Blood type | Optional |
| status | BOOLEAN | No | Person active status | Default true |
| created_at | DATETIME | No | Creation date | Not null |
| updated_at | DATETIME | Yes | Last update | Nullable |

**Indexes:**
- `uq_person_email_school` on `(school_id, email)` unique
- `idx_person_school` on `(school_id)`

### Table: User

| Field | Type | Nullable | Description | Constraints |
|-------|------|----------|-------------|-------------|
| user_id | UUID | No | User identifier | PK |
| person_id | UUID | No | Linked person | FK to `Person.person_id`, unique |
| username | VARCHAR | No | Login username | Unique |
| password_hash | VARCHAR | No | Password hash | Not null |
| authentication_type | VARCHAR | No | Mechanism used for login | Values: Local, Windows, External |
| status | BOOLEAN | No | Account enabled state | Default true |
| created_at | DATETIME | No | Account creation | Not null |
| updated_at | DATETIME | Yes | Last update | Nullable |
| last_access | DATETIME | Yes | Last successful login | Nullable |

**Indexes:**
- `uq_user_username` on `(username)` unique
- `uq_user_person` on `(person_id)` unique

### Table: User_Session

| Field | Type | Nullable | Description | Constraints |
|-------|------|----------|-------------|-------------|
| session_id | UUID | No | Session identifier | PK |
| user_id | UUID | No | Associated user | FK to `User.user_id` |
| start_date | DATETIME | No | Session start | Not null |
| end_date | DATETIME | Yes | Session end | Nullable |
| source_ip | VARCHAR | Yes | Client IP originating session | Optional |
| session_status | VARCHAR | No | Session lifecycle status | Values: Active, Closed |

**Indexes:**
- `idx_session_user` on `(user_id)`
- `idx_session_status` on `(session_status)`

---

## Relationships

- `Person.school_id` → `School.school_id`
- `User.person_id` → `Person.person_id`
- `User_Session.user_id` → `User.user_id`

> Cross-context references are represented by public IDs and resolved through APIs/events instead of direct FKs.

---

## Modeling decisions

- **Local ownership:** The Identity context owns `School`, `Person`, `User`, and `User_Session`.
- **External references:** `User` is referenced by public IDs from Authorization context, not by a direct cross-context FK.
- **Academic independence:** The person is not considered to be the owner of academic profiles such as Student or Teacher.

---

## Schema migration

**Tool:** Liquibase 4.29 (chained containers, ADR-010)
**Script location:** src/migrations/
**Naming convention:** V[NNN]__[description].sql

**Rollback policy:**
- Prefer additive migration with safe backfill.
- Roll back only when reversible and low-risk.

