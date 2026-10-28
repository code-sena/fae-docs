# ADR-027 — Attendance integrity: unique record per session/actor, optimistic locking, aggregate reports

- **ID:** ADR-027
- **Date:** 2026-10-26
- **Status:** Accepted
- **Authors:** FaceAttend EDU Team
- **Implementation note (verified 2026-10-26):** `05-ms-attendance` (`AttendanceRecordPersistenceAdapter`, `domain/port/out/*Repository`, `AttendanceRecordJpaRepository`), `database/05-ms-attendance-db`.

---

## Context

Two scans of the same learner (double tap, two kiosks) or two concurrent updates must not duplicate or silently overwrite attendance. Monthly reports must not trigger N+1 queries.

---

## Decision

**We decided:**
1. **Database-enforced uniqueness:** `uq_attendance_session_actor (class_session_id, academic_actor_id)`; a violation is translated to `DuplicateEntityException` (idempotent from the caller's point of view).
2. **Optimistic locking** via `row_version BIGINT DEFAULT 1` in every audited table (AGENTS §2.2, rule 12), incremented by trigger `fn_audit_timestamps()`.
3. **Hexagonal persistence:** domain port `AttendanceRecordRepository` (`domain/port/out`) implemented by `AttendanceRecordPersistenceAdapter`; controllers depend on use cases, never on Spring Data repositories.
4. **No cross-context joins:** actor/session data live in other schemas (no FK, ADR-002), so reports aggregate in-schema with JPQL `group by actor, status` and enrich in the application layer; there is no per-student `findById` loop.
5. **Mobile double submit:** capture buttons are disabled while `busy` (e.g. `RegisterFace.js`), preventing concurrent identical requests; the DB constraint is the real guard.
6. **Report export:** `GET /attendance-report?format=json|csv` (`text/csv`, `toCsv()`).

---

## Alternatives

| Alternative | Reason for discarding |
|---|---|
| Pessimistic lock per session | Contention with 30 learners scanning at once |
| Application-level "exists?" check only | Race condition |
| Cross-schema FK / join | Forbidden by ADR-002 |

---

## Consequences

- Correct under concurrency without locking reads.
- The SRS/jury asks for Excel/PDF "actas": **only CSV exists on the server today.** The web has `xlsx` (SheetJS) for import. Either produce `.xlsx` (Apache POI or SheetJS on web) or state CSV (opens in Excel) as the delivered format. Tracked in ADR-030.
- Tardiness tolerance (15 min) must be read from `academic_configuration` (`tardy_tolerance_minutes`); confirm the use case applies it before the demo.
