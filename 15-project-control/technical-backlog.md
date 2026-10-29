# Technical backlog

> Technical debt and improvements that "work but not as they should" (see `README.md` in this
> folder and `00-governance/definition-of-done.md`). Product user stories live in
> `03-product/product-backlog.md`. Code paths are relative to the `FULL` repository
> (`ProyectoFaceAttendEDU`).

| ID | Description | Impact if not resolved | Estimated effort | Priority | Sprint |
|----|-------------|----------------------|-----------------|---------|--------|
| TD-001 | Web dead code: `core/utils/timeParser.js`, `core/utils/attendanceUtils.js`, `core/utils/storage.js` and `services/api/justificationNotifications.js` are imported by nothing; unused helpers in `core/utils/dates.js`, `validation.js` and `formatting.js`. | Kept in the coverage denominator and read as if they were live code. | 1 SP | Low | — |
| TD-002 | Mobile dead or broken service methods: `CityService` (no `/api/v1/cities` route), `utils/codeGenerator.js` (unused, `Math.random`), and unused methods that send snake_case filters the Java/TS services ignore (`AttendanceService.getBySession/getByActor`, `ClassSessionService.getByScheduleBlock/getByDate`, `CourseService.getByProgram`, `SessionService.getByUserId`, …). Some are pinned on purpose by `webParityRequests.test.js` ("las peticiones existentes no cambian"). | A screen that starts using one gets wrong data silently (filter ignored). | 2 SP | Medium | — |
| TD-003 | Mobile school configuration stores `academicYear`, `startDate`, `endDate`, `maxAbsences`, `maxLatenesses` under its own keys; Web uses `web.period_start`, `web.period_end`, … Only the tardy tolerance is shared (`tardy_tolerance_minutes`). | A setting changed in one client is not seen by the other. | 2 SP | Medium | — |
| TD-004 | Node services (03, 07, 08, 09) generate no OpenAPI contract (`docs/openapi/README.md`), so `front-end/Web/src/api/__tests__/endpointsContract.test.js` cannot check their routes. | Broken links to those services are only found at run time. | 5 SP | Medium | — |
| TD-005 | Web session token in `sessionStorage` (audit S-10). Needs an HttpOnly cookie design (CORS, CSRF). | Exposed to any XSS in the web front-end. | 5 SP | High | — |
| TD-006 | No retention periods or scheduled deletion for attendance, justifications, supporting documents and match logs (audit L-NEW-03). The periods are a legal decision. | Personal data kept with no end date (Ley 1581). | 3 SP after the decision | High | — |
| TD-007 | Dashboard card "Instructores — presentes hoy" counts instructors whose taught/scheduled ratio is at least 85 %, not instructors who taught today. | The card label does not match what it measures. | 1 SP | Low | — |
| TD-008 | `FichaStorage` leaves `presentToday`, `lateToday`, `absentToday` at 0 (needs aggregation by date, not by actor). | Those ficha fields are always zero. | 2 SP | Low | — |
| TD-009 | Old `identity.user_session` rows still hold the token in clear text until purged after deploying N-04. | A leaked table copy still contains live tokens of old sessions. | 0.5 SP (ops) | High | — |
| TD-010 | Web/Mobile coverage gates measure the logic layers only (`coverageThreshold` in each `package.json`); screens under `view/` are covered by render and snapshot tests, without a floor. | Untested screen logic does not lower the reported number. | — | Low | — |
