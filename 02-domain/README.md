# 02 — Problem Domain

> The mental model of the business. It is not technology — it is understanding
> the problem the system solves before writing code. This section comes from Domain-Driven Design (DDD).

## Why this section exists

This section describes the domain model of the **FaceAttend EDU** project, understood as the business representation of the system. It defines the main concepts used by the educational institution to manage attendance, the entities involved in the process, the rules that must always be satisfied, and the events that occur during platform operation.

FaceAttend EDU should not be understood solely as an application with screens, forms, or records. From a domain perspective, the system revolves around concepts such as person, user, role, permission, school, program, cohort, environment, schedule, class session, attendance record, facial embedding, fingerprint embedding, justification, and alert.

The purpose of this section is to build a common language among the development team, the instructor, and project stakeholders so that everyone understands the business terms in the same way.

## Purpose

The domain section aims to:

- Identify the main bounded contexts of FaceAttend EDU.
- Define the business entities involved in attendance control.
- Establish the rules or invariants that the system must never violate.
- Describe the domain events that occur when a user interacts with the system.
- Serve as a basis for architecture, API, testing, and persistence design.
- Avoid ambiguity among terms such as person, user, cohort, environment, attendance, and justification.

## Scope of This Section

This section includes:

- Domain map and bounded contexts (8 contexts aligned with the data model).
- Business entities and their relationships.
- Business rules and invariants.
- Domain events and their side effects.
- Relationships among system concepts.
- Ubiquitous language shared by the team.

This section **does not include**:

- Database tables.
- Columns, foreign keys, or persistence schemas.
- API endpoints.
- User interface components.
- Decisions specific to a programming language or framework.

## Difference from the Data Section

This section describes the domain in business terms. It does not describe database tables, columns, or schemas. Those implementation details belong in the corresponding data folder.

Example of the difference:

| Approach | Question answered | Example |
|---|---|---|
| Domain | What does a cohort mean in the business? | A cohort represents a group of learners associated with a program, an academic period, and an instructor. |
| Data | How is that cohort stored in persistence? | It is stored in the `cohort` table with `cohort_id`, `program_id`, `academic_period_id`, `code`, and audit fields (no direct `school_id`). |

## Bounded Contexts

Based on the SRS and Event Storming, the FaceAttend EDU domain is organized into the following 8 bounded contexts (Audit retired in v1.1 — see history below):

| Context | Type | Main entities | Related requirements |
|---|---|---|---|
| Identity | Cross-cutting supporting | Person, App_User (`City` deprecated — external API) | RF 1, RNF 4 |
| Authorization | Cross-cutting supporting | Role, Permission, Role_Permission, User_Role | RF 1.1.1, RF 1.1.2, RF 4.6 |
| Academic | Core | School, Program, Academic_Period, Cohort, Course, Academic_Actor, Enrollment | RF 2, RF 8 |
| Scheduling | Core | Environment, Schedule_Block, Class_Session | RF 2.1, RF 2.3.2 |
| Attendance | System core | Attendance_Record, Justification_Type, Justification, Supporting_Document, Attendance_Report (derived) | RF 3, RF 6 |
| Biometric | Supporting (NoSQL + 1 SQL) | Facial_Embedding, Fingerprint_Embedding, Biometric_Update_Case (logical owner) | RF 3.1.1, RNF 4 |
| Notification | Supporting | Alert_Type, Alert | RF 4.1 |
| Configuration | Cross-cutting supporting | Academic_Configuration, Security_Configuration | RF 4 |

> **History:** Audit (Audit_Log, Error_Log) was removed as DDD bounded context in `refactor/remove-audit-domain`. Traceability is now a cross-cutting infrastructure concern (Kafka event log + centralized logging), not a business aggregate. RN-30/31/37-41/44/45 retired.

> **IoT:** `IoT_Device` / `Device_Assignment` (see `09-microservices/services/06-iot-service`) is a *technical* supporting context for future hardware, excluded from the core business map. It integrates only via `attendance-service` when devices are present.

> The **Identity** and **Authorization** contexts are cross-cutting because all other contexts depend on them to validate who the user is, which role they have, and which actions they may perform.

## Section Files

| File | Description | Question answered |
|---|---|---|
| [domain-map.md](./domain-map.md) | Domain map and bounded contexts. | What are the main business areas and how are they related? |
| [entities-and-rules.md](./entities-and-rules.md) | Entities, invariants, and business rules. | Which objects are important to the business and which rules must always be satisfied? |
| [domain-events.md](./domain-events.md) | Domain events and functional meaning. | Which facts occur within the system that are relevant to the business? |

## Relationship to the SRS

The information in this section is mainly based on the FaceAttend EDU Software Requirements Specification, especially on:

- RF 1: User management.
- RF 2: Environment management.
- RF 3: Generate entry and exit records.
- RF 4: System configuration.
- RF 5: Generate histories.
- RF 6: Justification management.
- RF 7: Advanced visualization and reporting.
- RF 8: School management.
- RNF 4: Security.

## Key Domain Concepts

The following terms must remain consistent (see `01-context/glossary.md` as source of truth and `domain-map.md#ubiquitous-language` for business definitions):

- Person / App_User / Credentials / Session (`City` deprecated — external API).
- Role / Permission / Role_Permission / User_Role (global roles, PK `(user_id, role_id)`).
- School / Program / Academic_Period / Cohort / Course / Academic_Actor / Academic_Actor_Type / Enrollment.
- Environment / Schedule_Block / Class_Session / Time_Block.
- Attendance_Record / Justification_Type / Justification / Supporting_Document / Attendance_Report / Recording_Method (FACIAL/MANUAL/IOT/IMPORT).
- Facial_Embedding / Fingerprint_Embedding / Biometric_Update_Case (logical owner Biometric).
- Alert_Type (severity/channel) / Alert.
- Academic_Configuration / Security_Configuration.

## Modeling Conventions

To maintain consistency within the domain documentation:

- Write entities with clear business names, for example: `Person`, `App_User`, `Cohort`, `Attendance_Record`.
- Write domain events in the past tense because they represent something that has already occurred, for example: `AttendanceRecorded`, `JustificationApproved`.
- Define invariants as mandatory rules, for example: "An academic_actor can have only one attendance_record per class_session."
- Avoid discussing tables, columns, or queries in this section.
- Keep the ubiquitous language aligned with the project's general glossary.

## Correlations

- Data model → `06-data/`
- Domain entities → `06-data/domains/`
- Architecture decisions → `05-architecture/decisions/records/`
- API contracts → `07-api/contracts/openapi/`
