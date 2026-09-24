# Problem Framing — Problem Definition

> **Why this document exists:** Before designing solutions, the team must be aligned on the problem it solves. This document captures that alignment.  
> A well-defined problem is already halfway to a solution.

---

## 1. The problem in one sentence

**Instructors, administrators, students, and educational institutions** who **manage training sessions and attendance in educational environments such as SENA training cohorts** struggle with **unreliable, manual, or inconsistent attendance validation** because **attendance depends on a single person, lacks automated identity verification, and does not have a centralized traceable workflow for records, histories, late arrivals, absences, and justifications**, resulting in **manual control of more than 30 users per instructor, inconsistent records, lost instructional time, and delayed follow-up on attendance issues**.

---

## 2. Affected users

| Segment | Description | Estimated size | Priority |
|---|---|---:|---|
| Instructor / Teacher | Responsible for taking attendance, monitoring group participation, identifying absences or late arrivals, and generating or reviewing attendance reports. | 1 instructor per cohort or training group; more than 30 learners per instructor according to SRS justification. Total number to be defined. | High |
| Student / Learner | Needs to register attendance quickly, consult personal attendance history, receive notifications, and justify absences or late arrivals when necessary. | More than 30 learners per instructor. Total number to be defined. | High |
| Administrator | Manages users, roles, permissions, environments, cohorts, schedules, reports, and general system configuration. | Small number per institution. Total number to be defined. | High |

### Jobs-to-be-done (JTBD)

#### Instructor / Teacher

**When** a training session starts or ends,  
**I want** attendance to be captured automatically, accurately, and without interrupting the class,  
**so that** I can focus on teaching and quickly identify absences, late arrivals, or participation issues.

---

#### Student / Learner

**When** I arrive at a training session,  
**I want** to register my attendance quickly and securely,  
**so that** my participation is recorded without delays, manual errors, or dependence on the instructor.

---

#### Student / Learner — Justification

**When** I miss a class or arrive late,  
**I want** to upload a justification and receive a clear result,  
**so that** my attendance history is fair, transparent, and updated according to institutional rules.

---

#### Administrator

**When** users, roles, environments, cohorts, schedules, or permissions change,  
**I want** to manage those changes centrally and securely,  
**so that** only authorized people can access or modify attendance information.

---

## 3. Evidence of the problem

> Because formal field research has not yet been documented, the current evidence is mainly documentary and comes from the SRS. Any interviews, observations, or support data must be added later without inventing findings.

| Evidence type | Source | Date | Key finding |
|---|---|---|---|
| Documentary analysis — Problem statement | SRS FaceAttend EDU, Section 1.a | 2025 | There is deficient validation of student attendance and a lack of effective follow-up in class participation, which may generate inconsistencies and affect instructor control. |
| Documentary analysis — Justification | SRS FaceAttend EDU, Section 1.c | 2025 | Delegating attendance registration of more than 30 users to one person, who also has other responsibilities, may be inefficient. Automation can reduce human error and improve monitoring. |
| Documentary analysis — Objectives | SRS FaceAttend EDU, Section 1.d | 2025 | The project seeks to validate attendance efficiently using technologies such as biometrics, mobile applications, QR codes, Wi-Fi, and AI, while minimizing inconsistencies and improving instructor control. |
| Documentary analysis — Use cases | SRS FaceAttend EDU, Section 4 | 2025 | Facial scanning may fail due to lighting, movement, camera issues, or unregistered faces. Alternative flows and justification management are needed. |
| Documentary analysis — Constraints and assumptions | SRS FaceAttend EDU, Sections 2.c and 2.d | 2025 | The system depends on internet access, functional cameras, compatible devices, institutional accounts, basic ICT skills, adequate lighting, and compliance with data-protection rules. |
| User interviews | Planned — interview guide in `04-requirements/HU` | 2026-10 (pilot cohort) | No formal interviews yet. 5 instructors + 30 learners target; instrument: semi-structured 20-min interview focused on JTBD §2. |
| Direct observation | Planned — shadowing protocol | 2026-10 | No shadowing yet. 3 sessions to be observed (manual roll call time, error rate, interruption duration). |
| Benchmarking | Initial desk research — 2026-09 | 2026-09-12 (internal) | Reviewed: SENA manual lists, generic biometric attendance vendors (ZKTeco), open-source face-api.js. Finding: manual consolidation >30 min/cohort; biometric vendors lack justification workflow. Gap: none integrates cohort+schedule+justification as FaceAttend does. |

### Assumptions requiring validation

The following assumptions are derived from the SRS and must be validated with real users or institutional stakeholders:

- Instructors currently spend significant time taking attendance manually.
- Manual attendance records generate inconsistencies or disputes.
- Students need a clearer way to consult attendance history and justify absences.
- Facial recognition is acceptable for users under institutional data-protection rules.
- Devices, cameras, lighting, and connectivity are sufficient for a pilot.
- The institution is willing to support implementation and training.
- Users have basic ICT skills.
- Biometric data can be processed with appropriate consent and security controls.

---

## 4. Current user solution (and its problems)

> How the user solves the problem today.

| Current solution | Limitations | Cost / Friction |
|---|---|---|
| Manual attendance list taken by the instructor | Slow, depends on one person, may interrupt class, error-prone, difficult to audit, and vulnerable to inconsistencies or impersonation. | Time lost per session. Exact value to be measured. |
| Paper records or informal spreadsheets | Difficult to consolidate, no real-time visibility, limited traceability, hard to filter by date, cohort, environment, or instructor. | Manual consolidation and increased risk of errors. |
| Informal justification process, such as verbal excuses, emails, or messages | No standardized workflow, no centralized evidence, no clear approval/rejection traceability. | Delayed resolution and lack of transparency. |
| Existing academic systems, if any | May not provide facial recognition, automated attendance capture, role-based dashboards, or justification workflows. Integration may be unavailable. | Depends on institutional context. To be validated. |

---

## 5. Solution hypothesis

> This is the first draft of the solution direction. It is not a commitment.

**We believe that** a web and mobile attendance-management system using facial recognition, with alternative verification methods for failed scans, role-based attendance histories, justification workflows, automatic notifications, and report dashboards  
**for** instructors, students, administrators, and supervisors in educational environments,  
**will achieve** more reliable attendance records, less manual workload for instructors, faster detection of absences or late arrivals, and greater transparency in justification processes.  
**We will know we succeeded when** the majority of expected attendance events are captured automatically and accepted without manual correction, and when instructors report that attendance management requires less time and fewer manual interventions.

---

## 6. Success metrics (North Star)

> Quantitative targets should be defined after a pilot or after collecting baseline data. Do not invent targets without evidence.

| Metric | Current baseline | 6-month target | How to measure it |
|---|---:|---:|---|
| % attendance automatically captured without correction | Manual (~0% auto) | 80% auto | System event logs vs roster; pilot reports |
| Avg time to register per student | ~45s manual (est. from SRS, to validate in pilot) | ≤10s FACIAL, ≤20s MANUAL | Timing tests in shadowing + `captured_at` deltas |
| % successful FACIAL under normal light | Not measured | ≥85% | Biometric service logs (confidence ≥ threshold) |
| % failed scans resolved via MANUAL/IMPORT | Not measured | ≥95% of failures | Alternative-method usage ratio |
| % justifications resolved inside system | Manual/offline (~10%) | ≥70% | Justification workflow logs |
| Time to generate attendance report (cohort/month) | ~30 min manual consolidation | ≤30s | Report generation logs + user testing |
| Inconsistencies/disputes per 100 records | Not measured (baseline in pilot) | −50% vs baseline | Support tickets + instructor feedback |
| SUS satisfaction (instructors/learners) | Not measured | ≥75 SUS | Post-pilot survey (n≥20) |

**North Star Metric:**  
Percentage of expected attendance events successfully captured automatically and accepted without manual correction.

---

## 7. Hypothesis risks

| Risk | Probability | Impact | Experiment to validate |
|---|---|---|---|
| Facial recognition fails due to lighting, movement, camera quality, or user positioning. | High | High | Run a controlled pilot, measure failed scans, test environmental conditions, and validate the alternative registration flow. |
| Users resist biometric capture or have privacy concerns. | Medium | High | Conduct informational sessions, obtain explicit consent, publish data-protection guidelines, and measure acceptance through surveys. |
| Connectivity or device limitations affect attendance capture. | Medium | High | Test devices, network conditions, camera availability, and error handling before pilot deployment. |
| The problem is not as frequent or painful as assumed. | Medium | Medium | Interview instructors, administrators, and learners; review current attendance practices; and validate with a small pilot. |
| Biometric data processing does not comply with data-protection requirements. | Low / Medium | High | Validate consent, encryption, access control, retention policies, and compliance with applicable regulations such as Colombian Law 1581 of 2012. |
| Scope grows beyond the core attendance problem, adding advanced analytics or integrations too early. | High | Medium | Maintain a clear roadmap, backlog prioritization, and explicit out-of-scope decisions. |
| Institutional roles and permissions are more complex than expected. | Medium | Medium | Validate role matrix with administrators and instructors before implementing fine-grained permissions. |

---

## 8. Out of scope (we do not solve)

> Explicitly define which related problems we are NOT solving in this version. This prevents scope creep.

- **Grading or academic-mark management:** FaceAttend EDU focuses on attendance, not academic evaluation.
- **Financial management, payments, or enrollment billing:** The product does not solve tuition, payments, or financial processes.
- **Physical access control:** The system does not control doors, turnstiles, or building-entry hardware.
- **QR-code or Wi-Fi-based attendance as primary methods:** The SRS mentions these technologies in objectives, but the current functional scope centers on facial recognition and manual/import registration.
- **IoT device management:** The IoT context is not part of the current data model. Device-related alerts are not included.
- **Advanced predictive analytics:** Dropout prediction, behavioral analytics, or AI-based academic-risk models are not included in the initial version.
- **External integrations not approved by the team:** Integration with institutional academic systems, identity providers, or communication platforms must be explicitly approved.
- **Multi-institution deployment before pilot validation:** The initial product should be validated in a controlled educational context before expanding.
- **Notification channels beyond the approved scope:** SMS, WhatsApp, or external messaging channels are not included unless explicitly approved. Initial notifications may be limited to in-app or email, depending on the product roadmap.

---

## Validation plan (post-evaluation improvement)

| Week | Activity | Owner | Output |
|---|---|---|---|
| 3 | 5 instructor interviews (JTBD validation) | PO | Notes + updated assumptions §3 |
| 4 | Shadowing 3 sessions (time/error measurement) | Tech Lead | Baseline metrics §6 |
| 5 | Benchmark matrix completed (3 systems) | Team | Benchmark doc in `99-archive` |
| 6 | Pilot cohort 1 (30 learners, FACIAL + MANUAL) | Team | Pilot report; adjust targets §6 |

## Correlations

- Product vision → `./vision.md`
- Product backlog, when created → `04-requirements/HU` + `./product-backlog.md`
- Context overview → `01-context/overview.md` / `01-context/glossary.md`
- Scope, assumptions, and constraints → `01-context/scope.md`
- Shared vocabulary → `01-context/glossary.md`
- Domain entities and business rules → `02-domain/entities-and-rules.md`
- Domain events → `02-domain/domain-events.md`
- Requirements source → SRS FaceAttend EDU (RF 1-8, RNF 4)
- Standards and references → IEEE Std 830-1998; Colombian Law 1581 of 2012