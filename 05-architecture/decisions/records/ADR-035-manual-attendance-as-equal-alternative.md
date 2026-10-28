# ADR-035 — Manual attendance is an equal alternative to biometrics

- **ID:** ADR-035
- **Date:** 2026-10-28
- **Status:** Accepted
- **Authors:** FaceAttend EDU Team
- **Related:** ADR-027 (one record per actor and session), ADR-029; `FULL` PR #297, #308 and #310.

---

## Context

Authorising biometric data is optional (Law 1581/2012 art. 6): refusing it must have no consequence. The API accepted `captureMethod = MANUAL`, but the Web kiosk could only create a record from a face or fingerprint match; the instructor could correct or remove a record, never create one. A learner without biometrics could not be marked present and became `Absent` when the class closed (finding L-NEW-01, critical). Separately, the automatic `Absent` records on class close only happen when ms-notification has a service account; without it nobody was ever marked absent (finding OPS-05).

---

## Decision

**We decided:**
1. The kiosk roster offers a **Presente** button next to every learner without a record, for instructors and administrators, with a confirmation. It creates the same record as a biometric match: `Present` (or `Late` past the school's tolerance), `captureMethod = MANUAL`, `matchScore = null`.
2. Reports, percentages and the at-risk alert (`ATTENDANCE_LOW`) count by **status only**, never by capture method, so a manual record weighs exactly like a biometric one.
3. When a class closes, learners without a record are marked `Absent` by ms-notification; this requires `NOTIFICATION_SERVICE_USERNAME/PASSWORD`, which are now a documented deployment requirement and, if missing, logged as a warning.

**Justification:** the legal guarantee has to exist where attendance is actually taken (the kiosk), not only in the API.

---

## Evaluated alternatives

| Alternative | Pros | Cons | Reason for discarding |
|---|---|---|---|
| Manual button in the kiosk roster — chosen | Same screen, same rules, auditable method | Easier to mark someone who is not there | — (chosen; the instructor is accountable and the method is recorded) |
| Separate "manual attendance" screen | Clear separation | Two flows to keep consistent | More code for the same result |
| Leave it to correct records after closing | No new UI | The learner is `Absent` until corrected; alerts may fire | Contradicts art. 6 in practice |

---

## Consequences

**Positive:** closes L-NEW-01; verified on the real stack: the manual learner stays `Present` after closing, the others become `Absent`, no alert is raised for them, and a correction to `Present`/`MANUAL` resolves an open at-risk alert.

**Negative / Trade-offs:** manual marking can be abused; mitigated by recording the method, the time and who did it.

**Impact on the system:** Web kiosk (`useAttendanceKioskViewModel.registerManually`, `AttendanceKioskView`), `08-ms-notification` (startup warning), `DESPLIEGUE.md`. Mobile does not take attendance and needs no change.

---

## Risks

| Risk | Probability | Impact | Mitigation |
|---|---|---|---|
| Notification deployed without the service account | Medium | High | Warning at startup; deployment checklist |
| Abuse of manual marking | Low | Medium | Method and author recorded; reports can filter by method |

---

## References

- Audit findings L-NEW-01 and OPS-05; `FULL` PR #297 (tests), #308 (warning and docs), #310 (kiosk button).
- Related to: ADR-027, ADR-029.
