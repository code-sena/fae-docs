# Normalization Assessment — FaceAttend EDU

> **Cut-off:** 2026-09-22 · **Record:** ADSO-3145556 · **Repo:** `code-sena/fae-docs` · Aligned to DBML v7
> **Scope:** 28 entities from `02-domain/entities-and-rules.md` against 29 SQL tables (PostgreSQL 17, `faceattend_db`, 8 schemas) + 2 MongoDB collections documented in `06-data/domains/`
> **Global verdict:** **3NF core with no intentional denormalizations; `attendance_report` is a derived JSONB projection outside the 3NF core.** Each business table is in 3NF; no transitive attributes or repeating groups. What looks like denormalization (logical cross-context ids without physical FK, `current_embedding_ref` SQL→NoSQL) is bounded-context isolation, not duplication.

---

## 1. Method

- Primary source: `06-data/domains/0*.md` (SQL DDL + dictionaries) and `02-domain/entities-and-rules.md` / `domain-map.md`.
- 1NF → 2NF → 3NF verified table by table with the classic criterion:
  - **1NF:** atomic domain per column, no lists or composite values, defined PK.
  - **2NF:** if composite PK → no non-key attribute depends on part of the PK.
  - **3NF:** no non-key attribute transitively depends on another non-key.
- BCNF reviewed only where non-trivial functional dependencies over a superkey exist (see §5).
- Activity and traceability cross-checked against `04-requirements` (39 HUs) and `09-microservices/service-catalog.md`.

---

## 2. Executive summary

| Bounded Context | Entities (02-domain) | SQL Tables / Collections | Verdict |
|---|---:|---|---|
| Identity | 2 (Person, App_User) + VOs | 4 tables (`person`, `app_user`, `user_session`, `password_policy`; `city` dropped) | 3NF |
| Authorization | 4 (Role, Permission, Role_Permission, User_Role) | 4 tables | 3NF |
| Academic | 8 (School, Program, Academic_Period, Cohort, Course, Academic_Actor_Type, Academic_Actor, Enrollment) | 8 tables | 3NF |
| Scheduling | 3 (Environment, Schedule_Block, Class_Session) | 3 tables | 3NF |
| Attendance | 5 (Attendance_Record, Justification_Type, Justification, Supporting_Document, Attendance_Report) | 5 tables (report = derived projection) | 3NF core + derived |
| Biometric | 3 (Facial_Embedding, Fingerprint_Embedding, Biometric_Update_Case) | 1 SQL table + 2 MongoDB collections | 3NF + document* |
| Notification | 2 (Alert_Type, Alert) | 2 tables | 3NF |
| Configuration | 2 (Academic_Configuration, Security_Configuration) | 2 tables (+ transient host of `biometric_update_case`) | 3NF |
| **Total** | **28 entities + 4 VOs** | **29 SQL + 2 Mongo** | **3NF core, 0 denormalizations** |

* Biometric embeddings are not assessed with relational normal forms; they are assessed as documents with external references by UUID/ObjectId (see §7).

**Historical count (2026-09-22 report):** the report stated "30/31 tables". DBML v7 recount is 29 SQL: `identity.city` removed (external API), Audit removed (platform observability), `attendance.attendance_report` added as derived projection.

> Entity↔table: `Facial_Embedding`/`Fingerprint_Embedding` deliberately have **no** SQL table — they live in MongoDB (`06-data/domains/06-biometric.md`). Not a gap: it is an engine decision documented in ADR-003. `Biometric_Update_Case` is logically owned by Biometric.

---

## 3. 1NF — Atomicity

**Verdict: met in all tables.**

- No column stores lists, CSV, unjustified JSON, or composite values.
- Time correctly modeled: `schedule_block.starts_at`/`ends_at` are atomic `TIME`; `academic_period.starts_on/ends_on` are `DATE`; `attendance_record.captured_at` is `TIMESTAMP`.
- Positive example: `enrollment` does not store `student_names[]`; `schedule_block` does not store `days VARCHAR "Mon,Wed,Fri"` — it uses one row per block with `day_of_week SMALLINT` + uniqueness constraints.
- Classic 1NF pitfall avoided: `person.document_number` split from `person.name/last_name` (no "full_name"), and `academic_actor.actor_code` separate from `school_id`.

---

## 4. 2NF — No partial dependencies

**Verdict: met.**

Only 2 tables have a composite PK and both are correctly in 2NF:

| Table | PK | Non-key attributes | Depends on full PK? |
|---|---|---|---|
| `role_permission` | `(role_id, permission_id)` | `assignment_date` + audit | Yes — the date belongs to the assignment, not to the role or permission alone |
| `user_role` | `(user_id, role_id)` | `assignment_date` + audit | Yes — same logic; no attribute describes only `user_id` or only `role_id` |

All other tables have a simple PK (SERIAL/BIGSERIAL/UUID) → 2NF is automatic once 1NF holds.

---

## 5. 3NF and BCNF — No transitive dependencies

**Verdict: 3NF in the 28-table core (+ `attendance_report` derived). BCNF also holds where applicable.**

Review by pattern:

### 5.1 Tables with no transitive risk (attributes depend only on the PK)

`role`, `permission`, `school`, `program`, `academic_period`, `cohort`, `course`, `academic_actor_type`, `environment`, `class_session`, `justification_type`, `supporting_document`, `alert_type`, `alert`, `academic_configuration`, `security_configuration` — all their attributes are facts about the PK, not about another non-key. No duplicated `school_name` in `program`. `school.city_name/district/country` are display fields from the external city API (`city` table removed), not duplicated relational data.

### 5.2 Tables with FKs that could be mistaken for transitivity

| Table | FKs | Transitive? | Rationale |
|---|---|---|---|
| `academic_actor` | `actor_type_id → academic_actor_type`, `school_id → school` | No | `actor_type_id` and `school_id` are FKs to distinct entities; no `academic_actor` attribute depends on `actor_type.name` or `school.name` |
| `enrollment` | `academic_actor_id`, `cohort_id` | No | `enrollment_status` depends on the enrollment `(actor,cohort)`, not on the actor or cohort alone |
| `schedule_block` | `cohort_id`, `course_id`, `environment_id`, `instructor_actor_id` | No | Each FK is an external identifier; schedules depend on the block, not on the course name |
| `attendance_record` | `class_session_id` (ext), `academic_actor_id` (ext) | No | `attendance_status`, `capture_method`, `match_score` depend on the record, not on the session |
| `justification` | `attendance_record_id` (UK), `justification_type_id` | No | `reason`, `review_status` depend on the justification, not on the type |
| `biometric_update_case` | `person_id` (ext), `current_embedding_ref` (Mongo logical) | No | `biometric_type`, `update_status` depend on the case |

### 5.3 Cases that *look like* denormalization but are not

- **`academic_configuration.school_id` and `scheduling.environment.school_id`:** they are **logical cross-context ids without physical FK** (single DB `faceattend_db`, one schema per context). Not enforcing the FK is a boundary decision, not a denormalization — it duplicates no data.
- **`city_id`/`city_name`/`district`/`country` in `school`:** `identity.city` was removed; these are an external catalog id plus display fields from `countriesnow.space`, not duplicated relational data.
- **`attendance_report.filters/result` (JSONB):** derived projection without FKs, outside the 3NF core by design.
- **Nullable FK ≠ denormalization:** `class_session.opened_by/closed_by` and `biometric_update_case.reviewed_by` may be NULL before the action. That is an optional relationship, not duplicated data.

### 5.4 BCNF

BCNF assessed on tables with non-trivial functional dependencies: `enrollment (academic_actor_id, cohort_id) → enrollment_id` does not apply because the PK is surrogate; the real FD is `enrollment_id → (actor, cohort)` and `enrollment_id` is a superkey → BCNF. `academic_actor (school_id, actor_type_id, actor_code)` has a UNIQUE over that triple, but the PK remains `academic_actor_id` (superkey) → BCNF.

---

## 6. Table-by-table matrix (3NF verdict)

| # | Table / Collection | PK | 1NF | 2NF | 3NF | Notes |
|---|---|---|---|---|---|---|
| 1 | `person` | `person_id` UUID | ✓ | ✓ | ✓ | `uq_person_document (document_type, document_number)`, no `school_id` |
| 2 | `app_user` | `user_id` UUID | ✓ | ✓ | ✓ | 1:1 with person, `username` UK |
| 3 | `user_session` | `session_id` UUID | ✓ | ✓ | ✓ | Session per user |
| 4 | `password_policy` | `policy_id` INT | ✓ | ✓ | ✓ | Global singleton — 1 effective row |
| 5 | `role` | `role_id` INT | ✓ | ✓ | ✓ | Global, no `school_id` |
| 6 | `permission` | `permission_id` INT | ✓ | ✓ | ✓ | `context.resource:action` |
| 7 | `role_permission` | `(role_id, permission_id)` | ✓ | ✓ | ✓ | M:N with `assignment_date` |
| 8 | `user_role` | `(user_id, role_id)` | ✓ | ✓ | ✓ | External `user_id`, no physical FK |
| 9 | `school` | `school_id` INT | ✓ | ✓ | ✓ | `code` UK, external `city_id` VARCHAR NULL |
| 10 | `program` | `program_id` INT | ✓ | ✓ | ✓ | `uq_program_code_school` |
| 11 | `academic_period` | `academic_period_id` INT | ✓ | ✓ | ✓ | `uq_period_name_school`, `chk_period_dates` |
| 12 | `cohort` | `cohort_id` BIGINT | ✓ | ✓ | ✓ | `code` UK |
| 13 | `course` | `course_id` INT | ✓ | ✓ | ✓ | `uq_course_code_program` |
| 14 | `academic_actor_type` | `actor_type_id` SMALLINT | ✓ | ✓ | ✓ | `STUDENT`/`INSTRUCTOR` |
| 15 | `academic_actor` | `academic_actor_id` BIGINT | ✓ | ✓ | ✓ | 2 UKs: `(school,type,code)` and `(person,type,school)` |
| 16 | `enrollment` | `enrollment_id` BIGINT | ✓ | ✓ | ✓ | UNIQUE `(actor,cohort)` |
| 17 | `environment` | `environment_id` INT | ✓ | ✓ | ✓ | `uq_environment_code_school` |
| 18 | `schedule_block` | `schedule_block_id` BIGINT | ✓ | ✓ | ✓ | 2 UNIQUE anti-double-booking + `chk_block_*` |
| 19 | `class_session` | `class_session_id` BIGINT | ✓ | ✓ | ✓ | UNIQUE `(block, date)` |
| 20 | `attendance_record` | `attendance_record_id` BIGINT | ✓ | ✓ | ✓ | UNIQUE `(session, actor)` |
| 21 | `justification_type` | `justification_type_id` INT | ✓ | ✓ | ✓ | Per-school + partial unique for globals |
| 22 | `justification` | `justification_id` BIGINT | ✓ | ✓ | ✓ | UNIQUE `attendance_record_id` (1:1) |
| 23 | `supporting_document` | `supporting_document_id` BIGINT | ✓ | ✓ | ✓ | FK → justification |
| 24 | `attendance_report` | `report_id` UUID | ✓ | ✓ | — | Derived JSONB projection, no FKs (outside 3NF core) |
| 25 | `biometric_update_case` | `case_id` UUID | ✓ | ✓ | ✓ | Logical owner Biometric; `chk_finger_number` |
| 26 | `alert_type` | `alert_type_id` SMALLINT | ✓ | ✓ | ✓ | `code` UK + severity/channel |
| 27 | `alert` | `alert_id` BIGINT | ✓ | ✓ | ✓ | `resolved_at` NULL = open |
| 28 | `academic_configuration` | `configuration_id` INT | ✓ | ✓ | ✓ | UNIQUE `(school, name)` |
| 29 | `security_configuration` | `configuration_id` INT | ✓ | ✓ | ✓ | Global UK `configuration_name` |
| 30 | `facial_embeddings` (Mongo) | `_id` ObjectId | — | — | — | Vector `encoding`, `is_active`, external `person_id` |
| 31 | `fingerprint_embeddings` (Mongo) | `_id` ObjectId | — | — | — | `finger_number` 1..10, `is_active` |

---

## 7. Intentional denormalizations

**None.**

> "As of this writing, the schema has no intentional denormalization — every table is in 3NF. This is stated plainly rather than manufactured content to fill the section." — same criterion as `energy-monitor` (reported as the cleanest of the cut).

If a denormalization is introduced in the future (e.g. caching `school_code` in `enrollment` to avoid JOIN, or `actor_code` in `alert`), it must be documented here with: (a) duplicated table/column, (b) reason (performance vs JOIN), (c) honesty mechanism (trigger, reconciliation job, or source `updated_at`), and (d) write cost.

---

## 8. NoSQL collections (Biometric)

`facial_embeddings` and `fingerprint_embeddings` are not normalized with relational 1NF-3NF. Their "normalized" equivalent design is:

- One document per embedding, no nested arrays breaking atomicity.
- Indexes `(person_id, is_active)` and `(person_id, finger_number, is_active)` enforce "at most one active per person/finger" at application level (RN-28), not via a Mongo unique constraint — documented as a domain invariant.
- `person_id` is an external reference to `Identity.person` — no DBRef or FK, resolved via events/API.
- `biometric_update_case.current_embedding_ref` (SQL) references `_id` (Mongo) as a logical `VARCHAR` — cross-paradigm, no FK.

---

## 9. Traceability and audit

- Standard audit fields on **all** SQL tables: `created_at`, `updated_at`, `deleted_at` (soft delete), `created_by`, `updated_by`, `deleted_by`, `row_version` (optimistic locking) — defined in `06-data/models.md` and replicated in each DDL.
- Audit context removed: observability is platform-provided, not relational tables.
- Logical-delete indexes: `WHERE deleted_at IS NULL` on high-volume tables (`person`, `academic_actor`, `enrollment`, `attendance_record`).

---

## 10. Open items

| ID | Description | Status |
|---|---|---|
| DATA-01 | Verify with real load that `uq_actor_person_type_school` does not block the "same person as STUDENT and INSTRUCTOR in two different schools" case | Pending integration test |
| DATA-02 | Decide whether `password_policy` moves to `configuration-service` in v5 (today lives in Identity for simplicity) | Open discussion — no normalization impact |

---

## 11. Conclusion

The FaceAttend EDU model is in **3NF core** with 29 SQL tables + 2 Mongo collections, traceable to 28 entities + 4 VOs in `02-domain/entities-and-rules.md`. No normalization debt remains. `attendance_report` is a derived projection outside the 3NF core by design. The only "divergence" flagged by the automatic detector (`Facial_Embedding` with no SQL table) is intentional and correct by design (MongoDB).

---

## Correlations

- Entities and rules → `02-domain/entities-and-rules.md`
- Context map → `02-domain/domain-map.md`
- Per-domain model (DDL) → `06-data/domains/0*.md`
- Cross-cutting summary → `06-data/models.md`
- Saga/Outbox patterns → `05-architecture/pattern-guide.md`
- API contracts → `07-api/contracts/openapi/`
- Service catalog and ownership → `09-microservices/service-catalog.md`, `09-microservices/data-ownership-matrix.md`
