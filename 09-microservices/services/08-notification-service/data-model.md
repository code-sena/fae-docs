# Data Model — Notification Service

> Authoritative owner of the alert catalog (`alert_type`) and the alert lifecycle (`alert`) that
> the Web/Mobile dashboards read.

---

## Database engine

**Engine:** PostgreSQL 17, schema `notification`
**Justification:** alerts are rows the dashboards list, filter and resolve; a transactional store
with soft delete and optimistic locking is enough — no event store needed.

---

## Schema

> Every table carries the mandatory audit block (`created_at`, `updated_at`, `deleted_at`,
> `created_by`, `updated_by`, `deleted_by`, `row_version DEFAULT 1`). Only business columns are
> listed. Source: `FULL/database/08-ms-notification-db/` (DDL + seed changesets).

### Table: `alert_type`

| Field | Type | Nullable | Description | Constraints |
|-------|------|----------|-------------|-------------|
| alert_type_id | SMALLINT | No | Catalog identifier | PK, auto-increment |
| code | VARCHAR | No | Machine code | Not null, UNIQUE |
| name | VARCHAR | No | Display name | Not null |
| severity | VARCHAR | No | `INFO` / `WARNING` / `CRITICAL` | Not null |
| channel | VARCHAR | No | `DASHBOARD` / `EMAIL` / `PUSH` | Default `DASHBOARD` |

**Indexes:** unique on `code` (catalog must not duplicate codes).

### Table: `alert`

| Field | Type | Nullable | Description | Constraints |
|-------|------|----------|-------------|-------------|
| alert_id | BIGINT | No | Alert identifier | PK, auto-increment |
| academic_actor_id | BIGINT | No | Subject → `academic.academic_actor` | Not null, **cross-context, no FK** |
| alert_type_id | SMALLINT | No | Type | Not null, **FK → alert_type** |
| raised_at | TIMESTAMP | No | When the alert was raised | Not null |
| resolved_at | TIMESTAMP | Yes | When it was resolved | NULL = open |

**Indexes:**
- `idx_alert_actor` `(academic_actor_id)`
- `idx_alert_type` `(alert_type_id)`

### Seeds (`02-dml/001-seed-alert-type-table.yaml`, 5 rows)

| code | name | severity | channel |
|------|------|----------|---------|
| ATTENDANCE_ABSENTEEISM | Recurrent absenteeism | WARNING | DASHBOARD |
| ATTENDANCE_TARDINESS | Repeated tardiness | WARNING | DASHBOARD |
| ATTENDANCE_LOW | Low attendance rate | CRITICAL | EMAIL |
| JUSTIFICATION_PENDING | Pending justification review | INFO | DASHBOARD |
| BIOMETRIC_UPDATE | Biometric update request | INFO | PUSH |

> Seeded rows are never edited — changes go in incremental changesets (rule in `AGENTS.md` §7).

---

## Relationships

- `alert.alert_type_id` → `alert_type.alert_type_id` (intra-context FK)
- `academic_actor_id` is a **cross-context public ID** (Academic) with no FK (see `AGENTS.md` §3)
- Alert generation itself is done by API clients (`POST /api/v1/alerts`) — this service does not
  compute absenteeism/tardiness (see `events.md`)

---

## Modeling decisions

- **Catalog + instance split:** severity/channel live on the type, resolution lives on the row, so
  adding a channel means editing data, not code.
- **Alerts are cheap rows:** no payload/document column — anything richer belongs to the owning
  context (attendance, biometric), referenced by `academic_actor_id` and `alert_type_id`.

---

## Schema migration

**Tool:** Liquibase 4.29, chained containers (ADR-010)
**Script location:** `FULL/database/08-ms-notification-db/` — `01-ddl` (schema + tables +
`003-enforce-updated-at-not-null`), `02-dml` (seed + sequence sync), `04-tcl`
(`fn_audit_timestamps` triggers), `05-rollbacks`.
**Naming convention:** `NNN-<action>-<object>-table.yaml`, additive, each with a rollback.

**Rollback policy:**
- Never delete alert history; resolve (`resolved_at`) instead.
- Seed changesets are idempotent (`ON CONFLICT DO NOTHING`).
