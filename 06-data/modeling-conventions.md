# Modeling Conventions

Naming and style conventions for every database in FaceAttend-Edu
(PostgreSQL 17, one schema per bounded context, plus MongoDB collections).
Source of truth: `database/CONVENCIONES.md` and the relational DBML
(`database/faceattend_edu_mr_v4.dbml`); where they disagree, **the DBML wins**.

## 1. Naming

### Tables

- **snake_case** everywhere: `academic_actor`, `schedule_block`,
  `biometric_update_case`.
- **Singular**: `person`, `course`, `enrollment` — never `persons`, `courses`.
- **Descriptive**: `attendance_record`, not a generic `attendance`.

### Columns

- **snake_case**: `person_id`, `actor_type_id`, `created_at`.
- Suffix **`_id`** for primary and foreign keys: `school_id`, `user_id`.
- Suffix **`_at`** for timestamps (`created_at`, `raised_at`) and **`_on`** for dates (`started_on`, `ended_on`, `enrolled_on`).
- Suffix **`_by`** for user references: `created_by`, `deleted_by`.
- Suffix **`_status`** for state fields: `session_status`,
  `enrollment_status`.

### Constraints

| Type | Format | Example |
|---|---|---|
| Primary key | `pk_{table}` | `pk_person` |
| Unique | `uq_{table}_{columns}` | `uq_person_document` |
| Foreign key | `fk_{table}_{reference}` | `fk_actor_school` |
| Index | `idx_{table}_{column}` | `idx_person_deleted` |

### Schemas

One PostgreSQL schema per bounded context:

| Context | Schema | Notes |
|---|---|---|
| Identity | `identity` | 4 tables |
| Authorization | `authorization` | 4 tables |
| Academic | `academic` | 8 tables |
| Scheduling | `scheduling` | 3 tables |
| Attendance | `attendance` | 5 tables |
| Biometric | `biometric` | Schema only (embeddings live in MongoDB) |
| Configuration | `configuration` | 3 tables (includes `biometric_update_case`) |
| Notification | `notification` | 3 tables (`alert_type`, `alert`, `notification_log`) |

### Liquibase files

| Type | Prefix | Example |
|---|---|---|
| Schema | `001-create-{schema}-schema.yaml` | `001-create-identity-schema.yaml` |
| Type/ENUM | `001-create-{type}-type.yaml` | `001-create-enrollment-status-type.yaml` |
| Table | `001-create-{table}-table.yaml` | `001-create-person-table.yaml` |
| Seed DML | `001-seed-{table}-table.yaml` | `001-seed-alert-type-table.yaml` |
| Trigger | `001-triggers-{function}.yaml` | `001-triggers-audit-timestamps.yaml` |
| Views | `001-create-{domain}-views.yaml` | `001-create-identity-views.yaml` |

Numeric prefixes (`001-`, `002-`, …) define execution order inside each phase.

## 2. Audit block

Every table carries this block **without exception** (per the DBML):

```
created_at   TIMESTAMP   NOT NULL
updated_at   TIMESTAMP   NOT NULL
deleted_at   TIMESTAMP   NULL
created_by   UUID        NULL
updated_by   UUID        NULL
deleted_by   UUID        NULL
row_version  BIGINT      NOT NULL DEFAULT 1
```

Rules:

- `deleted_at` → **soft delete**: no table deletes rows physically.
- `row_version` → **optimistic locking**, incremented on every UPDATE by the
  `fn_audit_timestamps()` trigger (one per table, e.g.
  `trg_person_audit_timestamps`): on INSERT it fills
  `created_at`/`updated_at`/`row_version` when null; on UPDATE it bumps
  `updated_at = NOW()` and `row_version = OLD.row_version + 1`.
- `created_by`/`updated_by`/`deleted_by` are always `UUID` (they point to
  `identity.app_user.user_id`).

## 3. Foreign keys

- **Intra-context** FKs are real database constraints
  (e.g. `academic.enrollment.cohort_id → academic.cohort`,
  `onDelete: CASCADE`).
- **Cross-context** references are **never** real FKs; they are documented in
  the column `remarks` ("Cross-context reference to Identity.person (NO
  FK)"), which keeps each schema independently migrable toward per-service
  databases.

## 4. Identifier strategy

| Type | Use | Example |
|---|---|---|
| `UUID` | Entities referenced from other contexts | `person_id`, `user_id`, `case_id` |
| `INT` autoincrement | Local entities, catalogs | `role_id`, `school_id` |
| `BIGINT` autoincrement | High-volume entities | `cohort_id`, `enrollment_id`, `alert_id` |
| `SMALLINT` autoincrement | Small catalogs | `actor_type_id`, `alert_type_id` |

Rule: a column that appears in a cross-context comment (`// field ->
Context.table`) must be `UUID`.

## 5. ENUMs

Enums are native PostgreSQL types (`CREATE TYPE ... AS ENUM`), not `VARCHAR`
with text validation (the DBML documents them as `varchar` with a
`note: ENUM`; the implementation uses native types):

```yaml
- createType:
    schemaName: attendance
    typeName: attendance_status
    asEnum: "Present;Absent;Late;Justified"
```

Columns reference the full type (`type: attendance.attendance_status`) and
carry a default. The model's ENUMs: `user_session_status`,
`authentication_type`, `enrollment_status`, `class_session_status`,
`attendance_status`, `capture_method`, `review_status`, `biometric_type`,
`update_status`.

## 6. Liquibase layout

Each context lives in `NN-ms-{domain}-db/` with this structure:

```
NN-ms-{domain}-db/
├── changelog/
│   └── changelog-master.yaml    ← entry point
├── 01-ddl/                      ← schemas, types, tables, views, functions, triggers
├── 02-dml/                      ← seeds, inserts, updates
├── 03-dcl/                      ← roles, grants, policies
├── 04-tcl/                      ← PL/pgSQL trigger functions
├── 05-rollbacks/                ← reversal scripts (always empty: `databaseChangeLog: []`)
└── docker-compose.yml           ← per-context execution
```

`changelog-master.yaml` executes the phases in order: `01-ddl` → `04-tcl` →
`02-dml` in most contexts (`06-ms-biometric-db` includes only `01-ddl`). Files use `relativeToChangelogFile: true` (the master, living in
`changelog/`, prefixes paths with `../`). For migration mechanics (no-edit
rule for applied changesets, seed immutability, Neon deploys) see
[Models](./models.md#6-migration-strategy).

## 7. Execution

- Full migration chain: `cd database && docker compose up -d` — starts
  PostgreSQL 17 (`faceattend_db` + the 8 schemas) and the 8 Liquibase services.
- Single context: `cd database/03-ms-academic-db && docker compose up`.
- Environment defaults: `POSTGRES_USER`/`POSTGRES_PASSWORD`/`POSTGRES_DB` =
  `postgres`/`postgres`/`faceattend_db`, port `5432`; Liquibase image
  `liquibase/liquibase:4.29.0`. PostgreSQL image: `postgres:17-alpine`.

## Open question

`database/CONVENCIONES.md` declares `updated_at` as **nullable**, while the
DBML declares it **not null**. Per the documented prelation rule the DBML
wins (and this document follows it), but the source file should be corrected
or the divergence explicitly accepted.
