# ADR-030 — Bulk import (client-parsed CSV/XLSX) and report export formats

- **ID:** ADR-030
- **Date:** 2026-10-26
- **Status:** Proposed (documents current behaviour and one correction)
- **Authors:** FaceAttend EDU Team
- **Implementation note (verified 2026-10-26):** `front-end/Web/src/components/common/importFileParser`, `ImportCoursesModal.js`, `services/api/staffProvisioning.js`; `05-ms-attendance/.../AttendanceReportController.java`.

---

## Context

The SRS requires bulk loading of users/courses and formal attendance reports. The jury asks whether a failing row rolls back the batch and which library produces Excel/PDF.

---

## Decision

**Current behaviour (as built):**
- Import parsing happens in the **web client** (SheetJS `xlsx` for `.xlsx`, delimiter detection for `.csv`, incl. the Spanish `;`). The UI previews rows, flags existing people (`markExistingStudents`: duplicate document/email) and the user confirms.
- Rows are created through the existing REST endpoints **one by one** (no backend batch endpoint); each service validates uniqueness (`uq_person_document`, `uq_person_email`). The batch is therefore **not atomic**: rows already created stay, failures are reported per row.
- Server export: attendance report as JSON or CSV only.

**We decided:** keep partial-success semantics with a per-row error report (all-or-nothing would block 99 valid learners for one typo), and state it explicitly. Export format for “acta” is CSV (opens in Excel); `.xlsx`/PDF are future work.

---

## Alternatives

| Alternative | Reason |
|---|---|
| Server `POST /import` with `@Transactional` all-or-nothing | Needs new endpoints in 4 services; poor UX on one bad row — **optional improvement** |
| Server validate-all then insert, return rejected rows | Best fit; not built |
| Apache POI / OpenPDF `.xlsx`/`.pdf` | Not present in any `pom.xml`; do not claim it |

---

## Consequences

- Answer honestly: parser = SheetJS on web; no Apache POI; partial import with report.
- **Corrections to consider:** (a) backend `POST /persons/import` with dry-run + atomic option; (b) `.xlsx` export via SheetJS on web (dependency already present; `xlsx 0.18.5` has known advisories — audit L-17, replace by `exceljs` or the CDN build); (c) letters A/F/J legend in the CSV.
