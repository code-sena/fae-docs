# Traceability Matrix

> Traceability connects every line of code to its business justification.
> It allows answering: "Why does this function exist?" and "Which HU covers this part of the system?"
> It also identifies: unimplemented requirements and code without a requirement (possible technical debt).

---

## How to use this matrix

```text
Requirement → HU → Test Case → Implementation → Service

If a requirement has no HU: it is not planned
If a HU has no test case: it has no completeness criterion
If a test case has no implementation: there is test technical debt
If there is code without an HU: possible gold-plating or bug introduced without a story
```

---

## FR → HU → Test → Service matrix

> **Status verified against the implemented back end (`../FULL`, services 01–09 + Kong gateway)
> on 2026-10-07.** ✅ means the flow exists in code and is reachable through Kong with an
> authenticated session; it does **not** mean automated tests exist — the test specs below are
> still *(proposed)* (the only implemented test file in the back end is
> `03-ms-academic/src/__tests__/academic.test.ts`, not mapped to an HU). NFR validation has not
> been executed.

### FR 1 — User Management

| FR ID | FR Description | HU(s) | Tests that verify it | Service | Status |
|-------|---------------|-------|---------------------|---------|--------|
| FR 1.1 | Register users with role assignment | HU-IAM-002 | `iam.user.register.spec.ts` *(proposed)* | identity-service | ✅ Done |
| FR 1.1.1 | Role assignment (user-person relationship) | HU-IAM-002 | `iam.role.assign.spec.ts` *(proposed)* | authorization-service | ✅ Done |
| FR 1.1.2 | Permission assignment by role on web platform | HU-IAM-003 | `iam.permissions.assign.spec.ts` *(proposed)* | authorization-service | ✅ Done |
| FR 1.2 | Register users via CSV files | HU-IAM-004 | `iam.user.csv-import.spec.ts` *(proposed)* | identity-service | 🔴 Pending |
| FR 1.3 | Login | HU-IAM-001 | `iam.auth.login.spec.ts` *(proposed)* | identity-service | ✅ Done |
| FR 1.4 | Password recovery | HU-IAM-005 | `iam.auth.recovery.spec.ts` *(proposed)* | identity-service | ✅ Done |
| FR 1.5 | Password change | HU-IAM-006 | `iam.auth.password-change.spec.ts` *(proposed)* | identity-service | 🔴 Pending |
| FR 1.6 | Activate/Deactivate users | HU-IAM-007 | `iam.user.toggle-status.spec.ts` *(proposed)* | identity-service | ✅ Done |
| FR 1.7 | Delete users | HU-IAM-008 | `iam.user.delete.spec.ts` *(proposed)* | identity-service | 🔴 Pending |
| FR 1.8 | User information query | HU-IAM-009 | `iam.user.query.spec.ts` *(proposed)* | identity-service | 🔴 Pending |

### FR 2 — Environment Management

| FR ID | FR Description | HU(s) | Tests that verify it | Service | Status |
|-------|---------------|-------|---------------------|---------|--------|
| FR 2.1 | Register environments/classrooms | HU-ACAD-001 | `acad.environment.register.spec.ts` *(proposed)* | academic-service | ✅ Done |
| FR 2.2 | Register cohorts/courses | HU-ACAD-002 | `acad.cohort.register.spec.ts` *(proposed)* | academic-service | ✅ Done |
| FR 2.3 | Assign responsible instructors | HU-ACAD-003 | `acad.responsible.assign.spec.ts` *(proposed)* | academic-service | ✅ Done |
| FR 2.3.1 | Assign cohorts to responsible users | HU-ACAD-004 | `acad.cohort.assign-to-responsible.spec.ts` *(proposed)* | academic-service | ✅ Done |
| FR 2.3.2 | Assign schedules/training blocks | HU-ACAD-005 | `acad.schedule.assign.spec.ts` *(proposed)* | academic-service | ✅ Done |

### FR 3 — Generate Entry and Exit Records

| FR ID | FR Description | HU(s) | Tests that verify it | Service | Status |
|-------|---------------|-------|---------------------|---------|--------|
| FR 3.1 | Register entries and exits | HU-ATT-001 | `att.attendance.entry-exit.spec.ts` *(proposed)* | attendance-service | ✅ Done |
| FR 3.1.1 | Facial registration | HU-ATT-001 | `att.attendance.facial.spec.ts` *(proposed)* | attendance-service / biometric-service | ✅ Done |
| FR 3.1.2 | Alternate registration for registered users with facial failure | HU-ATT-003 | `att.attendance.alternative.spec.ts` *(proposed)* | attendance-service | 🟡 In progress *(only `captureMethod=MANUAL`)* |
| FR 3.2 | Entry control for unregistered personnel | HU-ATT-004 | `att.nn.classification.spec.ts` *(proposed)* | attendance-service | 🔴 Pending |
| FR 3.2.1 | Alternative registration for NN users | HU-ATT-005 | `att.nn.alternative-register.spec.ts` *(proposed)* | attendance-service | 🔴 Pending |
| FR 3.x *(derived)* | Initial facial parameter enrollment | HU-ATT-002 | `att.biometric.enrollment.spec.ts` *(proposed)* | biometric-service | ✅ Done |

### FR 4 — System Configuration

| FR ID | FR Description | HU(s) | Tests that verify it | Service | Status |
|-------|---------------|-------|---------------------|---------|--------|
| FR 4.1 | Configure system alerts | HU-CONF-001 | `conf.alerts.configure.spec.ts` *(proposed)* | configuration-service | ✅ Done |
| FR 4.1.1 | Enable/Disable alerts | HU-CONF-002 | `conf.alerts.toggle.spec.ts` *(proposed)* | configuration-service | 🔴 Pending |
| FR 4.1.2 | Change notification tone | HU-CONF-003 | `conf.alerts.tone-change.spec.ts` *(proposed)* | configuration-service | 🔴 Pending |
| FR 4.2 | Configure application language | HU-CONF-004 | `conf.language.configure.spec.ts` *(proposed)* | configuration-service | 🔴 Pending |
| FR 4.3 | Update color palettes | HU-CONF-005 | `conf.color-palette.update.spec.ts` *(proposed)* | configuration-service | 🔴 Pending |
| FR 4.4 | Update user facial parameters | HU-CONF-006 | `conf.facial-param.update.spec.ts` *(proposed)* | biometric-service | ✅ Done |
| FR 4.5 | IoT device failure alerts management | HU-CONF-007 | `conf.iot-alerts.manage.spec.ts` *(proposed)* | configuration-service | 🔴 Pending *(only `IOT` capture value exists)* |
| FR 4.6 | Change user role | HU-IAM-010 | `iam.role.change.spec.ts` *(proposed)* | authorization-service | 🔴 Pending |
| FR 4.7 | Delays and justifications (Parameterization) | HU-CONF-008 | `conf.justification.parametrize.spec.ts` *(proposed)* | configuration-service | 🟡 In progress |

### FR 5 — Generate Historical Records

| FR ID | FR Description | HU(s) | Tests that verify it | Service | Status |
|-------|---------------|-------|---------------------|---------|--------|
| FR 5.1 | Attendance and absence query by parameters | HU-HIST-001 | `hist.attendance.query.spec.ts` *(proposed)* | attendance-service | 🔴 Pending |
| FR 5.2 | User change history | HU-HIST-002 | `hist.user-changes.query.spec.ts` *(proposed)* | identity-service (via platform logging) | 🔴 Pending *(only biometric embedding version history exists)* |

### FR 6 — Justification Management

| FR ID | FR Description | HU(s) | Tests that verify it | Service | Status |
|-------|---------------|-------|---------------------|---------|--------|
| FR 6.1 | Justification support upload | HU-JUS-001 | `jus.support.upload.spec.ts` *(proposed)* | attendance-service | ✅ Done |
| FR 6.2 | Justification approval and rejection | HU-JUS-002 | `jus.evaluation.approve-reject.spec.ts` *(proposed)* | attendance-service | ✅ Done |
| FR 6.3 | Automatic notification of result (A/R) | HU-JUS-003 | `jus.notification.result.spec.ts` *(proposed)* | attendance-service / notification-service | 🔴 Pending |

### FR 7 — Visualization and Advanced Reports

| FR ID | FR Description | HU(s) | Tests that verify it | Service | Status |
|-------|---------------|-------|---------------------|---------|--------|
| FR 7.1 | Export report | HU-REP-001 | `rep.export.report.spec.ts` *(proposed)* | attendance-service (derived projection) | 🔴 Pending |
| FR 7.2 | Role-based analytical dashboard | HU-REP-002 | `rep.dashboard.role-based.spec.ts` *(proposed)* | attendance-service (derived projection) | 🔴 Pending |
| FR 7.3 | Date-range reports | HU-REP-003 | `rep.export.date-range.spec.ts` *(proposed)* | attendance-service (derived projection) | 🔴 Pending |
| FR 7.4 | Delay and absence query by parameters | HU-REP-004 | `rep.delays-absences.query.spec.ts` *(proposed)* | attendance-service (derived projection) | 🔴 Pending |

### FR 8 — School Management

| FR ID | FR Description | HU(s) | Tests that verify it | Service | Status |
|-------|---------------|-------|---------------------|---------|--------|
| FR 8.1 | Add/Remove courses | HU-SCH-001 | `sch.course.add-remove.spec.ts` *(proposed)* | academic-service | 🔴 Pending |
| FR 8.2 | Subject study plan | HU-SCH-002 | `sch.study-plan.visualize.spec.ts` *(proposed)* | academic-service | 🔴 Pending |

---

## NFR → Validation matrix

| NFR ID | Description | How it is validated | Tool | Status |
|--------|-------------|-------------------|------|--------|
| NFR 1 | Performance: response < 5 seconds, load < 5 seconds | Load testing and response time measurement | k6 / JMeter *(to be defined)* | 🔴 Pending |
| NFR 2 | Availability/Usability: intuitive, accessible, multilanguage interface | Usability and accessibility testing (WCAG) | Manual testing + WCAG checklist | 🔴 Pending |
| NFR 3 | Compatibility: modern browsers, iOS and Android | Cross-browser and cross-device compatibility matrix | BrowserStack / manual testing *(to be defined)* | 🔴 Pending |
| NFR 4 | Security: authentication, encryption, HTTPS, Habeas Data | Penetration testing, static/dynamic analysis, security checklist | OWASP ZAP / SonarQube *(to be defined)* | 🔴 Pending |
| NFR 5 | Scalability: support 50% increase in users | Load and stress testing, capacity analysis | k6 / JMeter *(to be defined)* | 🔴 Pending |
| NFR 6 | Portability: functional from any browser, 24/7 | Cross-browser portability testing, uptime monitoring | Uptime monitoring *(to be defined)* | 🔴 Pending |

---

## Inverse traceability: HU → FR

| HU | Title | FR(s) it implements | Sprint |
|----|-------|---------------------|--------|
| HU-IAM-001 | User login | FR 1.3 | Sprint 1 |
| HU-IAM-002 | User registration with role assignment | FR 1.1, FR 1.1.1 | Sprint 1 |
| HU-IAM-003 | Role-based permission assignment | FR 1.1.2 | Sprint 1 |
| HU-IAM-004 | Massive user import via CSV | FR 1.2 | Sprint 2 |
| HU-IAM-005 | Password recovery | FR 1.4 | Sprint 2 |
| HU-IAM-006 | Password change | FR 1.5 | Sprint 2 |
| HU-IAM-007 | Activate/deactivate users | FR 1.6 | Sprint 2 |
| HU-IAM-008 | Delete users | FR 1.7 | Sprint 2 |
| HU-IAM-009 | User information query | FR 1.8 | Sprint 2 |
| HU-IAM-010 | Change user role | FR 4.6 | Sprint 2 |
| HU-ACAD-001 | Register environments/classrooms | FR 2.1 | Sprint 1 |
| HU-ACAD-002 | Register cohorts/courses | FR 2.2 | Sprint 1 |
| HU-ACAD-003 | Assign responsible instructors | FR 2.3 | Sprint 1 |
| HU-ACAD-004 | Assign cohorts to responsible users | FR 2.3.1 | Sprint 1 |
| HU-ACAD-005 | Assign jornadas/time blocks | FR 2.3.2 | Sprint 1 |
| HU-ATT-001 | Facial attendance registration | FR 3.1, FR 3.1.1 | Sprint 1 |
| HU-ATT-002 | Initial facial parameter enrollment | FR 4.4 *(related)* | Sprint 1 |
| HU-ATT-003 | Alternative registration for registered users | FR 3.1.2 | Sprint 1 |
| HU-ATT-004 | Entry control for unregistered/NN users | FR 3.2 | Sprint 2 |
| HU-ATT-005 | Alternative registration for NN users | FR 3.2.1 | Sprint 2 |
| HU-JUS-001 | Justification support upload | FR 6.1 | Sprint 2 |
| HU-JUS-002 | Justification approval and rejection | FR 6.2 | Sprint 2 |
| HU-JUS-003 | Automatic notification of justification result | FR 6.3 | Sprint 2 |
| HU-HIST-001 | Attendance and absence query by parameters | FR 5.1 | Sprint 2 |
| HU-HIST-002 | User change history | FR 5.2 | Sprint 3 |
| HU-REP-001 | Report export | FR 7.1 | Sprint 3 |
| HU-REP-002 | Role-based analytical dashboard | FR 7.2 | Sprint 3 |
| HU-REP-003 | Date-range reports | FR 7.3 | Sprint 3 |
| HU-REP-004 | Delay and absence query by parameters | FR 7.4 | Sprint 3 |
| HU-SCH-001 | Add or remove courses | FR 8.1 | Sprint 3 |
| HU-SCH-002 | Study plan visualization | FR 8.2 | Sprint 3 |
| HU-CONF-001 | Configure system alerts | FR 4.1 | Sprint 3 |
| HU-CONF-002 | Enable or disable alerts | FR 4.1.1 | Sprint 3 |
| HU-CONF-003 | Change notification tone | FR 4.1.2 | Sprint 3 |
| HU-CONF-004 | Configure application language | FR 4.2 | Sprint 3 |
| HU-CONF-005 | Update color palettes | FR 4.3 | Sprint 3 |
| HU-CONF-006 | Update user facial parameters | FR 4.4 | Sprint 2 |
| HU-CONF-007 | IoT device failure alerts management | FR 4.5 *(SRS index)* | Sprint 3 |
| HU-CONF-008 | Delay and justification parametrization | FR 4.7 | Sprint 2 |

---

## Status legend

| Status | Meaning |
|--------|---------|
| ✅ Done | Implemented in the current code base and reachable with a session (verified 2026-10-07). Automated tests are still *(proposed)* unless noted |
| 🟡 In progress | Partially implemented — see the note in the row |
| 🔴 Pending | Not implemented, or not verified as implemented |
| ⏸ Blocked | Has an external blocker |
| ❌ Cancelled | Removed from scope |

---

## Identified gaps (requirements without coverage)

> This section is updated automatically or manually when reviewing the matrix.
> A gap is: an FR without an HU, or an HU without a test, or a test without implementation.

| Gap type | Description | Required action | Owner | Date |
|----------|-------------|----------------|-------|------|
| SRS Inconsistency | FR 1.2 says 'Register users via CSV' but the description and output talk about login. | Clarify with instructor whether it is CSV or login. | Team / Instructor | 2026-06-16 |
| SRS Inconsistency | FR 1.3 says 'Login' but the input says 'Register in FaceAttend EDU'. | Correct description in SRS. | Team / Instructor | 2026-06-16 |
| SRS Inconsistency | FR 4.5 in the index says 'IoT Alerts' but in detail says 'Deactivate/reactivate users'. | Define actual scope with instructor. | Team / Instructor | 2026-06-16 |
| SRS Inconsistency | FR 4.6 is named 'Role Change' but the description talks about limiting access. | Correct description in SRS. | Team / Instructor | 2026-06-16 |
| SRS Inconsistency | NFR 2 is named 'Availability' but the criteria are about usability. | Recategorize or rename NFR. | Team / Instructor | 2026-06-16 |
| FR without dedicated HU | Use Cases 1-5 from the SRS have no independent HU (they are covered within existing HUs). | Verify coverage with the team. | Product Owner | 2026-06-16 |
| HU without test | No HU-level test spec is implemented; all specs in the matrix are *(proposed)*. The back end only contains `03-ms-academic/src/__tests__/academic.test.ts` (not mapped to an HU). | Execute the proposed specs. | QA / Dev | 2026-10-07 |
| NFR without validation tool | No NFR has a defined validation tool. | Define the testing and monitoring stack. | DevOps / Team | 2026-06-16 |

---

## How to maintain this matrix

1. When an HU is created: add the row in the FR → HU → Test → Service section.
2. When a test is written: note the file in the "Tests that verify it" column.
3. When an HU is completed: change the status to ✅.
4. At each Sprint Planning: review gaps and assign actions.

---

## Correlations

- User Stories → `04-requirements/user-stories.md`
- Non-Functional Requirements → `04-requirements/non-functional.md`
- Testing strategy → `11-quality/testing-strategy.md`
- DoD that determines when an HU is Done → `00-governance/definition-of-done.md`

---

## Project summary

| Metric | Value |
|---|---|
| **Total Functional Requirements (FR)** | 34 |
| **Total Non-Functional Requirements (NFR)** | 6 |
| **Total User Stories (HU)** | 39 |
| **FRs covered by HUs** | 34/34 ✅ |
| **FRs without coverage** | 0 |
| **FR rows implemented** | 18 ✅ · 2 🟡 · 21 🔴 (of the 41 rows above, verified 2026-10-07) |
| **HUs without tests** | 39 (test specs proposed; none implemented) |
| **SRS inconsistencies identified** | 5 |
| **Overall project status** | 🟡 Back end implemented (01–09 + Kong); tests and NFR validation pending |