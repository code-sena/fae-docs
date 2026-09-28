# Normalization Assessment — FaceAttend EDU

> **Cut-off:** 2026-09-22 · **Record:** ADSO-3145556 · **Repo:** `code-sena/fae-docs`
> **Scope:** 29 entities from `02-domain/entities-and-rules.md` against 31 SQL tables (PostgreSQL 17) + 2 MongoDB collections documented in `06-data/domains/`
> **Global verdict:** **3NF with no exceptions — no intentional denormalizations.** Each table is in 3NF; no transitive attributes or repeating groups. What looks like denormalization (external FKs without physical constraint, `current_embedding_ref` SQL→NoSQL) is bounded-context isolation, not duplication.

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
| Identity | 3 (Person, App_User, City) + VOs | 5 tables (`city`, `person`, `app_user`, `user_session`, `password_policy`) | 3NF |
| Authorization | 4 (Role, Permission, Role_Permission, User_Role) | 4 tables | 3NF |
| Academic | 8 (School, Program, Academic_Period, Cohort, Course, Academic_Actor_Type, Academic_Actor, Enrollment) | 8 tables | 3NF |
| Scheduling | 3 (Environment, Schedule_Block, Class_Session) | 3 tables | 3NF |
| Attendance | 4 (Attendance_Record, Justification_Type, Justification, Supporting_Document) | 4 tables | 3NF |
| Biometric | 2 (Facial_Embedding, Fingerprint_Embedding) | 2 MongoDB collections | N/A relational — normalized document by design* |
| Notification | 2 (Alert_Type, Alert) | 2 tables | 3NF |
| Audit | 2 (Audit_Log, Error_Log) | 2 tables | 3NF (append-only) |
| Configuration | 3 (Academic_Configuration, Security_Configuration, Biometric_Update_Case) | 3 tables | 3NF |
| **Total** | **29 entities + 4 VOs** | **31 SQL + 2 Mongo** | **3NF, 0 denormalizations** |

* Biometric is not assessed with relational normal forms; it is assessed as a document with external references by UUID/ObjectId (see §7).

**Historical count (2026-09-22 report):** the report stated "30 tables". The difference is `password_policy` (global singleton in Identity). Counting only multi-row business tables there are 30; including the singleton there are 31. Both figures are correct under their criterion — here the 31 are documented.

> Entity↔table: 29→29 perfect match except `Facial_Embedding`/`Fingerprint_Embedding` which deliberately have **no** SQL table — they live in MongoDB (`06-data/domains/06-biometric.md`). Not a gap: it is an engine decision documented in ADR-003.

---

## 3. 1NF — Atomicity

**Verdict: met in all tables.**

- No column stores lists, CSV, unjustified JSON, or composite values.
- Time correctly modeled: `schedule_block.starts_at`/`ends_at` are atomic `TIME`; `academic_period.starts_on/ends_on` are `DATE`; `attendance_record.captured_at` is `TIMESTAMPTZ`.
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

**Verdict: 3NF in all 31 tables. BCNF also holds where applicable.**

Review by pattern:

### 5.1 Tables with no transitive risk (attributes depend only on the PK)

`city`, `role`, `permission`, `school`, `program`, `academic_period`, `cohort`, `course`, `academic_actor_type`, `environment`, `class_session`, `justification_type`, `supporting_document`, `alert_type`, `alert`, `audit_log`, `error_log`, `academic_configuration`, `security_configuration` — all their attributes are facts about the PK, not about another non-key. No duplicated `school_name` in `program`, nor `city_name` in `school`; reference is by `school_id`/`city_id`.

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

- **`academic_configuration.school_id` and `scheduling.environment.school_id`:** they are **cross-context FKs without physical constraint** (external by UUID/SERIAL). Not enforcing the FK at engine level is a microservice isolation decision (Database per Service), not a denormalization — it duplicates no data.
- **`city_id` in `school`:** storing only the ID and not the city name is precisely the normalized form; duplicating `city_name` in `school` would be denormalization (and is not done).
- **Nullable FK ≠ denormalization:** `class_session.opened_by/closed_by` and `biometric_update_case.reviewed_by` may be NULL before the action. That is an optional relationship, not duplicated data.

### 5.4 BCNF

BCNF assessed on tables with non-trivial functional dependencies: `enrollment (academic_actor_id, cohort_id) → enrollment_id` does not apply because the PK is surrogate; the real FD is `enrollment_id → (actor, cohort)` and `enrollment_id` is a superkey → BCNF. `academic_actor (school_id, actor_type_id, actor_code)` has a UNIQUE over that triple, but the PK remains `academic_actor_id` (superkey) → BCNF.

---

## 6. Table-by-table matrix (3NF verdict)

| # | Table / Collection | PK | 1NF | 2NF | 3NF | Notes |
|---|---|---|---|---|---|---|
| 1 | `city` | `city_id` | ✓ | ✓ | ✓ | Simple catalog |
| 2 | `person` | `person_id` UUID | ✓ | ✓ | ✓ | `document_number` UK, no `school_id` (multi-tenant via actor) |
| 3 | `app_user` | `user_id` UUID | ✓ | ✓ | ✓ | 1:1 with person, `username` UK |
| 4 | `user_session` | `session_id` UUID | ✓ | ✓ | ✓ | Session per user |
| 5 | `password_policy` | `policy_id` SERIAL | ✓ | ✓ | ✓ | Global singleton — 1 effective row |
| 6 | `role` | `role_id` SERIAL | ✓ | ✓ | ✓ | Global, no `school_id` (v4 decision) |
| 7 | `permission` | `permission_id` SERIAL | ✓ | ✓ | ✓ |  |
| 8 | `role_permission` | `(role_id, permission_id)` | ✓ | ✓ | ✓ | M:N with `assignment_date` |
| 9 | `user_role` | `(user_id, role_id)` | ✓ | ✓ | ✓ | External `user_id`, no physical FK |
| 10 | `school` | `school_id` SERIAL | ✓ | ✓ | ✓ | `code` UK, external `city_id` |
| 11 | `program` | `program_id` SERIAL | ✓ | ✓ | ✓ | `uq_program_code_school` |
| 12 | `academic_period` | `academic_period_id` SERIAL | ✓ | ✓ | ✓ | `uq_period_name_school` |
| 13 | `cohort` | `cohort_id` BIGSERIAL | ✓ | ✓ | ✓ | `code` UK |
| 14 | `course` | `course_id` SERIAL | ✓ | ✓ | ✓ | `uq_course_code_program` |
| 15 | `academic_actor_type` | `actor_type_id` SMALLINT | ✓ | ✓ | ✓ | `STUDENT`/`INSTRUCTOR` |
| 16 | `academic_actor` | `academic_actor_id` BIGSERIAL | ✓ | ✓ | ✓ | 2 UKs: `(school,type,code)` and `(person,type,school)` |
| 17 | `enrollment` | `enrollment_id` BIGSERIAL | ✓ | ✓ | ✓ | UNIQUE `(actor,cohort)` |
| 18 | `environment` | `environment_id` SERIAL | ✓ | ✓ | ✓ | `uq_environment_code_school` |
| 19 | `schedule_block` | `schedule_block_id` BIGSERIAL | ✓ | ✓ | ✓ | 2 UNIQUE anti-double-booking (room, instructor) |
| 20 | `class_session` | `class_session_id` BIGSERIAL | ✓ | ✓ | ✓ | UNIQUE `(block, date)` |
| 21 | `attendance_record` | `attendance_record_id` BIGSERIAL | ✓ | ✓ | ✓ | UNIQUE `(session, actor)` |
| 22 | `justification_type` | `justification_type_id` SERIAL | ✓ | ✓ | ✓ | `requires_attachment` |
| 23 | `justification` | `justification_id` BIGSERIAL | ✓ | ✓ | ✓ | UNIQUE `attendance_record_id` (1:1) |
| 24 | `supporting_document` | `supporting_document_id` BIGSERIAL | ✓ | ✓ | ✓ | FK → justification |
| 25 | `alert_type` | `alert_type_id` SMALLINT | ✓ | ✓ | ✓ | `code` UK |
| 26 | `alert` | `alert_id` BIGSERIAL | ✓ | ✓ | ✓ | `resolved_at` NULL = open |
| 27 | `audit_log` | `audit_log_id` BIGSERIAL | ✓ | ✓ | ✓ | Append-only |
| 28 | `error_log` | `error_id` BIGSERIAL | ✓ | ✓ | ✓ | Append-only |
| 29 | `academic_configuration` | `configuration_id` SERIAL | ✓ | ✓ | ✓ | UNIQUE `(school, name)` |
| 30 | `security_configuration` | `configuration_id` SERIAL | ✓ | ✓ | ✓ | Global UK `configuration_name` |
| 31 | `biometric_update_case` | `case_id` UUID | ✓ | ✓ | ✓ | Conditional `biometric_type` + `finger_number` |
| 32 | `facial_embeddings` (Mongo) | `_id` ObjectId | — | — | — | Vector `encoding`, `is_active`, external `person_id` |
| 33 | `fingerprint_embeddings` (Mongo) | `_id` ObjectId | — | — | — | `finger_number` 1..10, `is_active` |

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
- `audit_log`/`error_log` are append-only (RN-44): never UPDATE/DELETE.
- Logical-delete indexes: `WHERE deleted_at IS NULL` on high-volume tables (`person`, `academic_actor`, `enrollment`, `attendance_record`).

---

## 10. Open items

| ID | Description | Status |
|---|---|---|
| DATA-01 | Verify with real load that `uq_actor_person_type_school` does not block the "same person as STUDENT and INSTRUCTOR in two different schools" case | Pending integration test |
| DATA-02 | Decide whether `password_policy` moves to `configuration-service` in v5 (today lives in Identity for simplicity) | Open discussion — no normalization impact |

---

## 11. Conclusion

The FaceAttend EDU model is in **3NF with no exceptions** with 31 SQL tables + 2 Mongo collections, 1:1 traceable to 29 entities + 4 VOs in `02-domain/entities-and-rules.md`. No normalization debt remains. The only "divergence" flagged by the automatic detector (`Facial_Embedding` with no SQL table) is intentional and correct by design (MongoDB).

---

## Correlations

- Entities and rules → `02-domain/entities-and-rules.md`
- Context map → `02-domain/domain-map.md`
- Per-domain model (DDL) → `06-data/domains/0*.md`
- Cross-cutting summary → `06-data/models.md`
- Saga/Outbox patterns → `05-architecture/pattern-guide.md`
- API contracts → `07-api/contracts/openapi/`
- Service catalog and ownership → `09-microservices/service-catalog.md`, `09-microservices/data-ownership-matrix.md`
