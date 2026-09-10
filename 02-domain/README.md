# 02 — Problem Domain

> The mental model of the business. It is not technology — it is understanding
> the problem the system solves before writing code. This section comes from Domain-Driven Design (DDD).

## Why this section exists

This section describes the domain model of the **FaceAttend EDU** project, understood as the business representation of the system. It defines the main concepts used by the educational institution to manage attendance, the entities involved in the process, the rules that must always be satisfied, and the events that occur during platform operation.

FaceAttend EDU should not be understood solely as an application with screens, forms, or records. From a domain perspective, the system revolves around concepts such as person, user, role, permission, school, program, cohort, environment, schedule, class session, attendance record, facial embedding, fingerprint embedding, justification, alert, and audit log.

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

- Domain map and bounded contexts (9 contexts aligned with the data model).
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
| Data | How is that cohort stored in persistence? | It is stored in the `cohort` table with `cohort_id`, `program_id`, `period_id`, `school_id`, and specific fields. |

## Bounded Contexts

Based on the data model, the FaceAttend EDU domain is organized into the following 9 bounded contexts:

| Context | Type | Main entities | Related requirements |
|---|---|---|---|
| Identity | Cross-cutting supporting | Person, App_User, City | RF 1, RNF 4 |
| Authorization | Cross-cutting supporting | Role, Permission, Role_Permission, User_Role | RF 1.1.1, RF 1.1.2, RF 4.6 |
| Academic | Core | School, Program, Academic_Period, Cohort, Course, Academic_Actor, Enrollment | RF 2, RF 8 |
| Scheduling | Core | Environment, Schedule_Block, Class_Session | RF 2.1, RF 2.3.2 |
| Attendance | System core | Attendance_Record, Justification_Type, Justification, Supporting_Document | RF 3, RF 6 |
| Biometric | Supporting (NoSQL) | Facial_Embedding, Fingerprint_Embedding | RF 3.1.1, RNF 4 |
| Notification | Supporting | Alert_Type, Alert | RF 4.1 |
| Audit | Supporting | Audit_Log, Error_Log | RF 5.2, RNF 4 |
| Configuration | Cross-cutting supporting | Academic_Configuration, Security_Configuration, Biometric_Update_Case | RF 4 |

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

The following important terms must remain consistent throughout this section:

- Person.
- App_User.
- Role.
- Permission.
- School.
- Program.
- Academic_Period.
- Cohort.
- Course.
- Academic_Actor.
- Enrollment.
- Environment.
- Schedule_Block.
- Class_Session.
- Attendance_Record.
- Justification.
- Supporting_Document.
- Facial_Embedding.
- Fingerprint_Embedding.
- Alert.
- Audit_Log.

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
