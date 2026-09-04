# 04 — Requirements

> **What is this?**  
> The formal specification of what FaceAttend EDU must do.  
> Functional requirements describe **what the system does**.  
> Non-functional requirements describe **how well it does it**.

---

## Why this section exists

Requirements are the contract between the team and the client/stakeholder.

Without them:

- There is no way to verify whether the system is complete.
- Scope changes have no baseline for comparison.
- Tests have no success criterion.
- The team cannot prove that the SRS functional and non-functional requirements were implemented.

For **FaceAttend EDU**, this section is especially important because the system handles:

- Attendance records.
- Student and instructor information.
- Facial recognition or biometric parameters.
- Justifications for absences or late arrivals.
- Role-based reports and dashboards.
- Sensitive personal data subject to data-protection rules.

---

## Requirement baseline

This section is based on the Software Requirements Specification of **FaceAttend EDU**.

The main requirement sources are:

| Source | Description |
|---|---|
| RF 1 | User management, roles, permissions, authentication, recovery, activation, deletion, and user queries. |
| RF 2 | Environment management, cohorts/courses, responsible users, and schedules. |
| RF 3 | Entry and exit registration, facial recognition, alternative registration, and unregistered-user control. |
| RF 4 | System configuration, alerts, languages, appearance, facial-parameter updates, role changes, and justification parameters. |
| RF 5 | Attendance and absence history, and user-change history. |
| RF 6 | Justification support upload, approval, rejection, and automatic notification. |
| RF 7 | Advanced reports, dashboards, date-range reports, and delay/absence queries. |
| RF 8 | School management, courses, and study plans. |
| RNF 1-6 | Performance, availability/usability, compatibility, security, scalability, and portability. |
| Use cases 1-5 | Facial attendance registration, facial scan, justification registration, face registration, and user consultation. |
| Constraints and assumptions | Internet access, functional cameras, compatible devices, institutional accounts, basic ICT skills, lighting conditions, and data-protection compliance. |

---

## Types of requirements

### Functional requirements

Functional requirements describe **what the system does**: functions, behaviors, and data transformations.

Project example:

> The system must allow a registered student to record attendance using facial recognition.

> The system must allow an instructor to consult attendance records by cohort, environment, date, or instructor.

> The system must allow a student to upload a justification for an absence or late arrival.

---

### Non-functional requirements

Non-functional requirements describe **how well the system does it**: performance, availability, security, compatibility, scalability, and portability.

Project example:

> The system must respond to user requests in less than 5 seconds.

> The system must protect sensitive biometric and personal data during storage and transmission.

> The system must be compatible with modern web browsers and mobile operating systems such as Android and iOS.

Non-functional requirements are often harder to verify and are frequently ignored. **They are equally important.**

---

## Files in this section

| File | Description | Key question answered | Status |
|---|---|---|---|
| [functional.md](./functional.md) | Complete list of functional requirements derived from the SRS. | What must FaceAttend EDU do for each user role? | 🟡 Pending completion |
| [non-functional.md](./non-functional.md) | Performance, availability, security, compatibility, scalability, and portability requirements. | How well must the system perform and protect data? | 🟡 Pending completion |
| [user-stories.md](./user-stories.md) | Formalized user stories derived from the product backlog or SRS requirements. | What user needs must be satisfied and how will they be accepted? | 🔴 Pending |
| [traceability-matrix.md](./traceability-matrix.md) | Matrix connecting requirements, user stories, and test cases. | Is every requirement covered by tests? | 🔴 Pending |
| `_template-hu.md` | Template for user stories with acceptance criteria. | How should user stories be written? | 🔴 Optional |
| `_template-nfr.md` | Template for non-functional requirements with metrics and verification methods. | How should NFRs be specified and verified? | 🔴 Optional |

---

## Requirement identification conventions

To keep traceability with the SRS and future tests, the following prefixes are proposed:

| Prefix | Meaning | Example |
|---|---|---|
| FR | Functional requirement | FR-ATT-001 |
| NFR | Non-functional requirement | NFR-SEC-001 |
| HU | User story, when created | HU-ATT-001 |
| TC | Test case | TC-ATT-001 |

### Functional module codes

| Module code | Module | SRS source |
|---|---|---|
| USR | User and access management | RF 1 |
| ACAD | Academic structure | RF 2, RF 8 |
| ATT | Attendance capture | RF 3 |
| CONF | System configuration | RF 4 |
| HIST | History | RF 5 |
| JUS | Justifications | RF 6 |
| REP | Reports and dashboards | RF 7 |
| SCH | School management | RF 8 |

---

## Functional requirements summary

The complete detailed list must be documented in `functional.md`. The following table summarizes the functional areas of FaceAttend EDU.

| Module | SRS source | Internal prefix | Main required capabilities | Priority |
|---|---|---|---|---|
| User and access management | RF 1.1 - RF 1.8 | FR-USR | Register users, assign roles, assign permissions, import users by CSV, login, recover password, change password, activate/deactivate users, delete users, and query user information. | High / Medium |
| Academic structure | RF 2.1 - RF 2.3.2 | FR-ACAD | Register environments or classrooms, register cohorts or courses, assign responsible users, assign cohorts to responsible users, and assign training schedules or time blocks. | High |
| Attendance capture | RF 3.1 - RF 3.2.1 | FR-ATT | Register entry and exit, validate facial parameters, allow alternative registration when facial scan fails, control unregistered users, and register casual or NN users. | High |
| System configuration | RF 4.1 - RF 4.7 | FR-CONF | Configure alerts, notification tones, languages, color palettes, facial-parameter updates, user activation/deactivation, role changes, and valid justification parameters. | Medium / High |
| History | RF 5.1 - RF 5.2 | FR-HIST | Query attendance and absence records by date, cohort, environment, instructor, or person, and query user-change history. | High / Medium |
| Justifications | RF 6.1 - RF 6.3 | FR-JUS | Upload justification support, approve or reject justifications, and automatically notify the result. | High |
| Reports and dashboards | RF 7.1 - RF 7.4 | FR-REP | Export reports, display role-based dashboards, generate date-range reports, and query delays or absences by parameters. | High / Medium |
| School management | RF 8.1 - RF 8.2 | FR-SCH | Add or delete courses and view study plans, schedules, instructors, and cohort specifications. | High |

---

## Non-functional requirements summary

The complete detailed list must be documented in `non-functional.md`. The following table summarizes the non-functional requirements identified in the SRS.

| Category | Proposed ID | Requirement | Metric / Acceptance condition | How to verify | SRS source |
|---|---|---|---|---|---|
| Performance | NFR-PERF-001 | The system must respond to user requests quickly. | Response time less than 5 seconds. | Functional tests, load testing, or manual timing validation. | RNF 1 |
| Performance | NFR-PERF-002 | The system must load pages quickly. | Page load time less than 5 seconds. | Browser performance test or Lighthouse-style validation. | RNF 1 |
| Performance | NFR-PERF-003 | The system must recover from interface failures without losing information. | Recovery in less than 5 minutes without data loss. | Failure-recovery test. | RNF 1 |
| Availability / Usability | NFR-AVAIL-001 | The interface must be easy to learn and navigate. | Users complete basic flows without critical errors. | Usability test or pilot observation. | RNF 2 |
| Availability / Usability | NFR-AVAIL-002 | The interface must be clear and readable. | Text, buttons, and states are understandable. | Usability review or accessibility checklist. | RNF 2 |
| Availability / Usability | NFR-AVAIL-003 | The system must tolerate user errors. | Users receive clear error messages and can recover. | Negative-flow tests. | RNF 2 |
| Compatibility | NFR-COMP-001 | The system must work in modern browsers. | Chrome, Firefox, Edge, Safari, Brave, Vivaldi, or equivalent. | Compatibility matrix tests. | RNF 3, RNF 6 |
| Compatibility | NFR-COMP-002 | The system must work on mobile devices. | Android and iOS latest supported versions. | Mobile device testing. | RNF 3 |
| Compatibility | NFR-COMP-003 | The interface must adapt to different screens. | Desktop, laptop, tablet, and mobile resolutions. | Responsive design tests. | RNF 3 |
| Security | NFR-SEC-001 | Users must authenticate before accessing the system. | No protected function is accessible without login. | Authentication tests. | RNF 4 |
| Security | NFR-SEC-002 | Sensitive data must be protected. | Personal and biometric data encrypted during storage and transmission. | Security review and configuration validation. | RNF 4 |
| Security | NFR-SEC-003 | Communication must use secure protocols. | HTTPS required for client-server communication. | Deployment and network validation. | RNF 4 |
| Security | NFR-SEC-004 | The system must comply with data-protection rules. | Compliance with applicable regulations such as Colombian Law 1581 of 2012 and, if applicable, GDPR principles. | Consent, privacy policy, and data-handling checklist. | SRS objectives, RNF 4 |
| Scalability | NFR-SCAL-001 | The system must support user growth. | Support a potential 50% increase in users within less than one year without significant degradation. | Load and stress testing. | RNF 5 |
| Scalability | NFR-SCAL-002 | The system must handle sudden traffic peaks. | No significant downtime or errors during peak load. | Stress test or simulated peak. | RNF 5 |
| Portability | NFR-PORT-001 | The system must be accessible from any supported browser. | Core flows work regardless of supported browser. | Cross-browser tests. | RNF 6 |
| Portability | NFR-PORT-002 | The system must remain available. | Proposed uptime target: 99.9% or higher, to be validated with the team. | Monitoring or pilot availability log. | RNF 6 |

> Some SRS metrics are examples or need validation. Targets such as 99.9% uptime, 12-minute maximum downtime, or 50% growth must be confirmed with the team and the instructor before being treated as formal commitments.

---

## Functional requirement example format

When completing `functional.md`, each functional requirement should follow a format similar to this:

```markdown
| ID | Module | Description | Source | Priority |
|----|--------|-------------|--------|---------|
| FR-USR-001 | User and access management | The system must allow user registration and role assignment. | SRS RF 1.1, RF 1.1.1 | High |
| FR-ATT-001 | Attendance capture | The system must allow a registered user to record entry and exit. | SRS RF 3.1 | High |
| FR-ATT-002 | Attendance capture | The system must validate facial parameters against registered users. | SRS RF 3.1.1 | High |
| FR-JUS-001 | Justifications | The system must allow a student to upload justification support. | SRS RF 6.1 | High |
| FR-REP-001 | Reports and dashboards | The system must display a role-based dashboard. | SRS RF 7.2 | High |