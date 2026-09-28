# Project Glossary

> **Instructions:** Define here all technical and business terms used in the project.
> This is the official dictionary — if there is ambiguity, this document wins.
> Add terms throughout the project, not only at the start.

---

## How to use this glossary

1. Before using a technical or business term in code, docs, or conversations: look it up here.
2. If it's not there: add it with its definition.
3. If there is disagreement about the definition: discuss it as a team and update this document.

---

## Domain terms — FaceAttend EDU Ubiquitous Language

| Term | Definition | Notes / Synonyms |
|------|-----------|-----------------|
| Person | Natural person registered in the institution; root for identity; uniqueness is `(document_type, document_number)` | Not to confuse with App_User |
| App_User | Login account (username + password hash) bound 1:1 to Person; owns credentials and sessions | Syn: User (avoid generic) |
| City | DEPRECATED — cities come from external `countriesnow.space` API; `School.city_id` is VARCHAR NULL with no FK | Do not recreate as table |
| School | Educational institution (multi-tenant root) that groups programs, cohorts, environments | Formerly `Institution` — avoid |
| Program | Academic program (e.g., Software Development) within a School | Not Cohort |
| Academic_Period | Time division (e.g., 2024-Q1) that scopes cohorts and enrollments | Syn: Term, Semester |
| Cohort | Group of learners (ficha) associated with Program + Period + School + Instructor | Core unit for attendance; not Course |
| Course | Subject/module within a Program (e.g., Mathematics I) | Not Cohort |
| Academic_Actor | Unified representation of Person as Student or Instructor within academic context | Replaces separate Student/Teacher tables |
| Academic_Actor_Type | Catalog: Student, Instructor | — |
| Enrollment | Links Academic_Actor (Student) to Cohort | — |
| Environment | Physical space (classroom, lab) where classes are held, belonging to a School | — |
| Schedule_Block | Recurring time slot (day + start/end) for Cohort+Course+Instructor+Environment | Prevents double-booking |
| Class_Session | Concrete occurrence of a Schedule_Block on a specific date; attendance attaches here | Status: Open/Closed/Cancelled |
| Attendance_Record | Proof of presence for one Academic_Actor in one Class_Session (Present/Absent/Late/Justified, FACIAL/MANUAL/IOT/IMPORT) | Central business entity |
| Justification_Type | Per-school catalog of absence reasons (`school_id` NULL = global) with requires_attachment flag | Partial unique for globals |
| Justification | Request 1:1 with Attendance_Record to explain absence; lifecycle Pending→Approved/Rejected + resolution_notes | Not Supporting_Document |
| Supporting_Document | File evidence attached to Justification (MIME, URI) | — |
| Attendance_Report | Derived reporting projection (filters + result JSONB, no FKs), outside 3NF core | Not a business aggregate |
| Facial_Embedding | Face vector stored in MongoDB for recognition; one active per Person | Not photo |
| Fingerprint_Embedding | Fingerprint template (1..10 per Person) | — |
| Alert_Type / Alert | Catalog + system-generated notification; Alert_Type has severity (INFO/WARNING/CRITICAL) and channel (DASHBOARD/EMAIL/PUSH) | Not Notification channel |
| Academic_Configuration | Per-school academic parameters (e.g., tardy_tolerance_minutes) | — |
| Security_Configuration | Global security parameters (e.g., max_login_attempts) | Password policy lives in Identity |
| Biometric_Update_Case | Workflow for facial/fingerprint re-enrollment (Pending→In_Review→Approved/Rejected); logical owner Biometric | Unified FACIAL/FINGERPRINT; `chk_finger_number` |
| Credentials / Session | Value objects: login data and authenticated temporary state | No identity outside App_User |
| Time_Block / Recording_Method | Value objects: day+time interval and FACIAL/MANUAL/IOT/IMPORT | `day_of_week` 1..7 ISO |

---

## Technical terms of the project

| Term | Definition |
|------|-----------|
| Microservice | Independent service with a single responsibility, its own process, and its own database |
| Domain Event | A fact that occurred in the business that other services can observe. Name always in past tense. |
| Bounded Context | Boundary within which a particular domain model has consistent meaning |
| API Gateway | Single entry point to the system that routes requests to the corresponding microservices |
| Circuit Breaker | Pattern that stops calls to a failing service, preventing failure cascades |
| Saga | Sequence of local transactions across different services with compensating transactions on failure |
| Dead Letter Queue | Queue where messages that could not be processed after several retries are sent |
| Idempotence | Property of an operation to produce the same result if executed multiple times |

---

## Acronyms

| Acronym | Meaning |
|---------|---------|
| IAM | Identity and Access Management |
| JWT | JSON Web Token |
| API | Application Programming Interface |
| CRUD | Create, Read, Update, Delete |
| DTO | Data Transfer Object |
| FR | Functional Requirement |
| NFR | Non-Functional Requirement |
| SLO | Service Level Objective |
| SLA | Service Level Agreement |
| ADR | Architecture Decision Record |
| PR | Pull Request |
| DoD | Definition of Done |
| CI/CD | Continuous Integration / Continuous Delivery |
