# Data Model — Scheduling Service

> Authoritative owner of rooms (`environment`), the recurring weekly grid (`schedule_block`) and
> its materialization into concrete `class_session` rows — the entity Attendance writes against.

---

## Database engine

**Engine:** PostgreSQL 17, schema `scheduling`
**Justification:** the anti-double-booking rules (room slot, instructor slot) and session uniqueness
are enforced with database unique constraints, not with application-level locking.

---

## Schema

> Every table carries the mandatory audit block (`created_at`, `updated_at`, `deleted_at`,
> `created_by`, `updated_by`, `deleted_by`, `row_version DEFAULT 1`) — see `database/CONVENCIONES.md`.
> Only business columns are listed below. Source of truth: `database/faceattend_edu_mr_v4.dbml` +
> `FULL/database/04-ms-scheduling-db/`.

### Enum: `class_session_status` (native PG type)

| Value |
|-------|
| Open |
| Closed |
| Cancelled |

### Table: `environment`

| Field | Type | Nullable | Description | Constraints |
|-------|------|----------|-------------|-------------|
| environment_id | INT | No | Room/lab identifier | PK, auto-increment |
| school_id | INT | No | Owning school → `academic.school` | Not null, **cross-context, no FK** |
| code | VARCHAR | No | Room code | Not null |
| name | VARCHAR | No | Display name | Not null |
| capacity | SMALLINT | No | Seat capacity | Not null |
| status | BOOLEAN | No | Active flag | Default true |

**Constraints / indexes:**
- `uq_environment_code_school` UNIQUE `(school_id, code)`

### Table: `schedule_block`

| Field | Type | Nullable | Description | Constraints |
|-------|------|----------|-------------|-------------|
| schedule_block_id | BIGINT | No | Block identifier | PK, auto-increment |
| cohort_id | BIGINT | No | Cohort → `academic.cohort` | Not null, **cross-context, no FK** |
| course_id | INT | No | Course → `academic.course` | Not null, **cross-context, no FK** |
| environment_id | INT | No | Room | Not null, **FK → environment** |
| instructor_actor_id | BIGINT | No | Instructor → `academic.academic_actor` | Not null, **cross-context, no FK** |
| day_of_week | SMALLINT | No | ISO weekday | `chk_block_day_of_week (1..7)` |
| starts_at | TIME | No | Slot start | Not null |
| ends_at | TIME | No | Slot end | `chk_block_time (starts_at < ends_at)` |

**Constraints / indexes:**
- `uq_block_environment_slot` UNIQUE `(environment_id, day_of_week, starts_at)` — no room double-booking
- `uq_block_instructor_slot` UNIQUE `(instructor_actor_id, day_of_week, starts_at)` — no instructor double-booking
- `idx_block_cohort`, `idx_block_course`

### Table: `class_session`

| Field | Type | Nullable | Description | Constraints |
|-------|------|----------|-------------|-------------|
| class_session_id | BIGINT | No | Session identifier | PK, auto-increment |
| schedule_block_id | BIGINT | No | Materialized block | Not null, **FK → schedule_block** |
| session_date | DATE | No | Concrete date | Not null |
| opened_by | BIGINT | Yes | Instructor who opened it → `academic_actor` | **Cross-context, no FK** |
| opened_at | TIMESTAMP | Yes | Opening timestamp | Nullable |
| closed_by | BIGINT | Yes | Instructor who closed it | **Cross-context, no FK** |
| closed_at | TIMESTAMP | Yes | Closing timestamp | Nullable |
| session_status | `class_session_status` | No | Lifecycle state | Not null |

**Constraints / indexes:**
- `uq_session_block_date` UNIQUE `(schedule_block_id, session_date)` — one session per block and date
- `idx_session_date`

---

## Relationships

- `schedule_block.environment_id` → `environment.environment_id` (intra-context FK)
- `class_session.schedule_block_id` → `schedule_block.schedule_block_id` (intra-context FK)
- `school_id`, `cohort_id`, `course_id`, `instructor_actor_id`, `opened_by`, `closed_by` are
  **cross-context public IDs without FK**, validated by the application (see `AGENTS.md` §2.1/§3).

---

## Modeling decisions

- **Anti-double-booking lives in the DDL:** uniqueness on room and instructor slots is a pair of
  UNIQUE constraints, so two concurrent requests cannot race past a check-then-act validation.
- **Materialization over derivation:** `class_session` rows are created explicitly (unique per
  block + date) instead of being computed from `schedule_block` on the fly, giving attendance a
  stable target and giving open/close auditing a place to live.
- **Attendance owns nothing here:** the status machine (`Open → Closed | Cancelled`) is scheduling's;
  Attendance only reads the session and writes `attendance_record`.

---

## Schema migration

**Tool:** Liquibase 4.29, chained containers (ADR-010)
**Script location:** `FULL/database/04-ms-scheduling-db/` — `changelog/changelog-master.yaml`
includes `01-ddl` (schema, `class_session_status` type, three tables), `02-dml`, `03-dcl`,
`04-tcl` (audit trigger `fn_audit_timestamps`), `05-rollbacks`.
**Naming convention:** `NNN-<action>-<object>-table.yaml` — additive changesets, each with a rollback.

**Rollback policy:**
- Never drop or rewrite `class_session` history; only additive changes to the session flow.
- `05-rollbacks/changelog.yaml` carries the down path per changeset.
