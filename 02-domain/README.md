# 02 — Problem Domain

> **What is this?** The mental model of the business. It is not technology — it is understanding
> the problem the system solves before writing code. This section comes from Domain-Driven Design (DDD).

## Why this section exists

This section describes the domain model of the **FaceAttend EDU** project, understood as the business representation of the system. It defines the main concepts used by the educational institution to manage attendance, the entities involved in the process, the rules that must always be satisfied, and the events that occur during platform operation.

FaceAttend EDU should not be understood solely as an application with screens, forms, or records. From a domain perspective, the system revolves around concepts such as user, person, role, permission, cohort, environment, schedule, facial record, attendance, late arrival, absence, justification, supporting document, approval, history, report, and notification.

The purpose of this section is to build a common language among the development team, the instructor, and project stakeholders so that everyone understands the business terms in the same way.

## Purpose

The domain section aims to:

- Identify the main subdomains of FaceAttend EDU.
- Define the business entities involved in attendance control.
- Establish the rules or invariants that the system must never violate.
- Describe the domain events that occur when a user interacts with the system.
- Serve as a basis for architecture, API, testing, and persistence design.
- Avoid ambiguity among terms such as user, person, cohort, environment, attendance, late arrival, absence, and justification.

## Scope of This Section

This section includes:

- Domain map and main contexts.
- Business entities.
- Business rules and invariants.
- Domain events.
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
| Domain | What does a cohort mean in the business? | A cohort represents a group of learners associated with a responsible user, a schedule, and an environment. |
| Data | How is that cohort stored in persistence? | It is stored in a structure or table with identifiers, relationships, and specific fields. |

## Planned Subdomains

Based on the SRS, the FaceAttend EDU domain can initially be organized into the following subdomains:

| Subdomain | Preliminary description | Related requirements |
|---|---|---|
| Security, Identity, and Access | Manages users, credentials, roles, permissions, activation, deactivation, password recovery, auditing, and sensitive-data protection. | RF 1, RF 4.6, RF 5.2, RNF 4 |
| Academic Structure | Includes environments, cohorts, courses, schedules, responsible users, instructors, and curriculum. | RF 2, RF 8 |
| Attendance Recording | Covers facial scanning, alternative records, entries, exits, unregistered users, late arrivals, and absences. | RF 3 |
| Justification Management | Manages supporting-document uploads, superior review, approval, rejection, and result notification. | RF 6, RF 4.7 |
| Histories and Reports | Enables consultation of histories, dashboards, date-range reports, and filters by cohort, environment, instructor, or user. | RF 5, RF 7 |
| Configuration and Alerts | Includes notification, language, color-palette, user-parameter, and system-alert configuration. | RF 4 |

> The **Security, Identity, and Access** subdomain is considered cross-cutting because other subdomains depend on it to validate who the user is, which role they have, and which actions they may perform.

## Section Files

| File | Description | Question answered | Status |
|---|---|---|---|
| [domain-map.md](./domain-map.md) | Domain map and main contexts. | What are the main business areas and how are they related? | 🟡 |
| [entities-and-rules.md](./entities-and-rules.md) | Entities, invariants, and business rules. | Which objects are important to the business and which rules must always be satisfied? | 🟡 |
| [domain-events.md](./domain-events.md) | Domain events and functional meaning. | Which facts occur within the system that are relevant to the business? | 🟡 |

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
- Use cases related to facial recording, facial scanning, absence justification, face registration, and user consultation.

## Key Domain Concepts

The following important terms must remain consistent throughout this section:

- User.
- Person.
- Role.
- Permission.
- Cohort.
- Environment.
- Schedule.
- Responsible user.
- Instructor.
- Student.
- Attendance record.
- Late arrival.
- Absence.
- Justification.
- Supporting document.
- Facial parameter.
- NN user.
- Dashboard.
- History.
- Report.

## Modeling Conventions

To maintain consistency within the domain documentation, the following is recommended:

- Write entities with clear business names, for example: `User`, `Person`, `Cohort`, `Environment`, `Schedule`, `AttendanceRecord`, `Justification`.
- Write domain events in the past tense because they represent something that has already occurred, for example: `AttendanceRegistered`, `LateArrivalDetected`, `JustificationApproved`.
- Define invariants as mandatory rules, for example: “A user cannot have more than one entry record for the same schedule.”
- Avoid discussing tables, columns, or queries in this section.
- Keep the ubiquitous language aligned with the project's general glossary.
- Validate with the team any term specific to the SENA context, such as cohort, environment, or instructor.

## Current Status

The domain section is currently being defined. The detail files are pending completion, but the overall structure is based on the functional and non-functional requirements defined in the SRS.

| Document | Status |
|---|---|
| [domain-map.md](./domain-map.md) | 🟡 In progress |
| [entities-and-rules.md](./entities-and-rules.md) | 🟡 In progress |
| [domain-events.md](./domain-events.md) | 🟡 In progress |
