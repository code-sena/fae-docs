# Entities, Value Objects, and Business Rules

> **What to fill in here:** The building blocks of the domain following the DDD tactical model.
> This document translates domain knowledge (obtained in Event Storming) into code models.

> **Stack note:** The concepts of Entity, Value Object, and Aggregate are language-independent.
> Code examples (classes, interfaces, decorators) are written in pseudo-TypeScript
> to illustrate the idea. To see the implementation in your technology:
> [`_stacks/node-typescript.md`](../_stacks/node-typescript.md) ·
> [`_stacks/java-spring.md`](../_stacks/java-spring.md) ·
> [`_stacks/python-fastapi.md`](../_stacks/python-fastapi.md) ·
> [`_stacks/go.md`](../_stacks/go.md)

---

## Tactical DDD concepts

This document defines the main entities in the FaceAttend EDU domain, their value objects, and the business rules that the system must guarantee at all times.

The goal is for the team to share a common language about business concepts and for the rules to support functional design, testing, and implementation.

## Conventions

- **Entity:** business concept with its own identity and lifecycle. Example: a Cohort or an Attendance Record.
- **Value object:** concept without its own identity, described by its attributes. Example: a time range or a facial parameter.
- **Invariant:** rule that must always be satisfied within the domain.
- Rules use the `RN` prefix (Business Rule) and relate to SRS requirements.

---

## Entity Map by Context

| Context | Main entities | Value objects |
|---|---|---|
| Security, Identity, and Access | Person, Access Account, Role, Permission | Credentials, Session |
| Academic Structure | School, Course, Cohort, Environment, Schedule, Responsible User | Time Block |
| Attendance Recording | Attendance Record, Recording Device | Facial Parameter, Recording Method |
| Justification Management | Justification, Evaluation | Supporting Document, Justification Criterion |
| Histories and Reports | Attendance History, Report | Query Filter |
| Configuration and Alerts | User Configuration, Alert | Notification Preference |

---

## Main Entities

### Person

**Context:** Security, Identity, and Access.

**Description:** Represents a real individual in the system: student, teacher, administrator, or supervisor. It is the central entity of the academic and attendance domains.

**Identity:** Unique person identifier.

**Conceptual attributes:**
- First and last names.
- Document or institutional identifier.
- Email address.
- Telephone number.
- Status (active/inactive).
- Relationship with the school.

**Lifecycle:** Registered → Active → Inactive.

**Associated rules:** RN-01, RN-02, RN-03.

### Access Account

**Context:** Security, Identity, and Access.

**Description:** Represents the credentials that allow a Person to access the system. A Person may exist without an active Account, but an Account cannot exist without an associated Person.

**Identity:** Unique account identifier.

**Conceptual attributes:**
- Username.
- Status (active/inactive).
- Last access date.
- Associated role.

**Lifecycle:** Created → Active → Inactive.

**Associated rules:** RN-04, RN-05, RN-06.

### Role

**Context:** Security, Identity, and Access.

**Description:** Defines an account's access level. The roles planned in the SRS are:
- Supervisor.
- Administrator.
- Teacher / Instructor.
- Student.
- Parent or guardian, if applicable.

**Associated rules:** RN-07, RN-08, RN-09.

### Permission

**Context:** Security, Identity, and Access.

**Description:** Specific action that a role may perform in the system, such as registering users, approving justifications, exporting reports, or consulting histories.

**Associated rules:** RN-09, RN-10.

### School

**Context:** Academic Structure.

**Description:** Educational institution that uses FaceAttend EDU. It groups courses, cohorts, environments, people, and records.

**Conceptual attributes:**
- Name.
- Institutional identifier.
- Address.
- Telephone number.
- Email address.
- Status.

**Associated rules:** RN-11, RN-12.

### Course

**Context:** Academic Structure.

**Description:** Program or training plan to which cohorts, subjects, or academic activities may be associated.

**Conceptual attributes:**
- Course name.
- Course code.
- Associated school.
- Status.

**Associated rules:** RN-12, RN-13.

### Cohort

**Context:** Academic Structure.

**Description:** Group of students associated with a responsible user, a schedule, and an environment. It is the main unit for attendance control.

**Conceptual attributes:**
- Cohort identifier.
- Associated course or program.
- School.
- Assigned responsible user.
- Assigned schedule.
- Assigned environment.
- Status.

**Lifecycle:** Registered → Active → Finished.

**Associated rules:** RN-14, RN-15, RN-16.

### Environment

**Context:** Academic Structure.

**Description:** Physical space where training takes place and attendance is recorded.

**Conceptual attributes:**
- Name.
- School.
- Status.

**Associated rules:** RN-15, RN-17.

### Schedule / Training Block

**Context:** Academic Structure.

**Description:** Time range assigned for a cohort to be under the responsibility of a teacher or instructor. It defines when students are expected to record entry and exit.

**Conceptual attributes:**
- Date.
- Start time.
- End time.
- Associated cohort.
- Associated responsible user.
- Associated environment.

**Associated rules:** RN-18, RN-19, RN-20.

### Responsible User

**Context:** Academic Structure.

**Description:** Teacher or instructor responsible for a cohort during a specific schedule.

**Associated rules:** RN-16, RN-21.

### Attendance Record

**Context:** Attendance Recording.

**Description:** Event evidencing a student's entry or exit during a specific schedule. It is the central business entity of FaceAttend EDU.

**Conceptual attributes:**
- Student.
- Schedule.
- Movement type: entry or exit.
- Date and time.
- Recording method: facial, alternative, or manual.
- Status: present, late, absent, or justified.

**Lifecycle:** Recorded → Validated → Justified if applicable.

**Associated rules:** RN-22, RN-23, RN-24, RN-25, RN-26.

### Facial Parameter

**Context:** Attendance Recording / Security.

**Description:** Biometric representation associated with a Person, used to validate facial recognition.

**Conceptual attributes:**
- Associated person.
- Model or capture version.
- Active/inactive status.
- Update date.

**Associated rules:** RN-27, RN-28, RN-29.

### Recording Device

**Context:** Attendance Recording.

**Description:** Physical or logical device from which attendance is recorded, such as a mobile device, camera, or IoT device assigned to an environment.

**Conceptual attributes:**
- Identifier.
- Associated environment.
- Status: active, inactive, or maintenance.
- Last connection.

**Associated rules:** RN-30, RN-31.

### Justification

**Context:** Justification Management.

**Description:** Request submitted by a student to explain an absence or late arrival. It may have an attached supporting document and must be evaluated by a superior.

**Conceptual attributes:**
- Associated attendance record.
- Reason.
- Supporting document.
- Status: pending, approved, or rejected.
- Evaluating user.
- Request date.
- Evaluation date.

**Lifecycle:** Pending → Approved / Rejected.

**Associated rules:** RN-32, RN-33, RN-34, RN-35, RN-36.

### Supporting Document

**Context:** Justification Management.

**Description:** Document or evidence uploaded by the student to support a justification.

**Conceptual attributes:**
- Supporting-document type.
- Description.
- Upload date.
- Upload status.

**Associated rules:** RN-33, RN-34.

### Attendance History

**Context:** Histories and Reports.

**Description:** Set of attendance, late-arrival, and absence records available for consultation and analysis.

**Associated rules:** RN-37, RN-38.

### Report

**Context:** Histories and Reports.

**Description:** Result of a structured query over system histories. It may be viewed or exported according to the user's role.

**Conceptual attributes:**
- Report type.
- Date range.
- Applied filters.
- Requesting user.

**Associated rules:** RN-38, RN-39.

### User Configuration

**Context:** Configuration and Alerts.

**Description:** User's personal preferences within the system.

**Conceptual attributes:**
- Language.
- Enabled or disabled alerts.
- Notification tone.
- Color palette, if applicable.

**Associated rules:** RN-40, RN-41.

### Alert

**Context:** Configuration and Alerts.

**Description:** Notification generated by the system for relevant events, such as scan failures, device failures, late arrivals, absences, or justification results.

**Conceptual attributes:**
- Alert type.
- Receiving user.
- Status: read/unread.
- Generation date.

**Associated rules:** RN-42, RN-43.

---

## Value Objects

### Credentials

**Context:** Security, Identity, and Access.

**Description:** Set of data required to authenticate an account. It has no identity of its own outside the account.

**Associated rules:** RN-04, RN-44.

### Session

**Context:** Security, Identity, and Access.

**Description:** Temporary state indicating that a user has been authenticated and may interact with the system for a limited period.

**Associated rules:** RN-04, RN-45.

### Time Block

**Context:** Academic Structure.

**Description:** Time interval consisting of a date, start time, and end time. It is used for schedules, reports, and filters.

**Associated rules:** RN-18, RN-19.

### Recording Method

**Context:** Attendance Recording.

**Description:** Indicates how attendance was recorded. Planned values:
- Facial.
- Alternative through data validation.
- Manual or alternative for unregistered users.

**Associated rules:** RN-25, RN-26.

### Justification Criterion

**Context:** Justification Management.

**Description:** Definition of which types of excuse or supporting document are valid for justifying an absence or late arrival. It is configured by the administrator.

**Associated rules:** RN-34, RN-46.

### Query Filter

**Context:** Histories and Reports.

**Description:** Set of parameters used to query information. Planned values: date, cohort, environment, instructor, student, and date range.

**Associated rules:** RN-37, RN-38.

---

## Business Rules

### Security, Identity, and Access

| ID | Rule | Related requirement |
|---|---|---|
| RN-01 | Every Access Account must be associated with a valid Person. | RF 1.1, RF 1.1.1 |
| RN-02 | A Person may exist without an active Account, but there cannot be active academic records without an associated Person. | RF 1.6, RF 1.7 |
| RN-03 | A Person cannot be deleted if they have active associated attendance records, cohorts, or justifications. | RF 1.7 |
| RN-04 | Every user must authenticate before accessing system functions. | RF 1.3, RNF 4 |
| RN-05 | Credentials must be stored and transmitted securely. | RNF 4 |
| RN-06 | An administrator may activate or deactivate accounts without deleting associated historical information. | RF 1.6 |
| RN-07 | Every Account must have a defined role. | RF 1.1.1 |
| RN-08 | An account's role may be changed only by a user with administrative permissions. | RF 4.6 |
| RN-09 | Access permissions must depend on the assigned role. | RF 1.1.2 |
| RN-10 | A user cannot execute an action without permission associated with their role. | RF 1.1.2 |

### Academic Structure

| ID | Rule | Related requirement |
|---|---|---|
| RN-11 | All academic data must belong to a School. | RF 8 |
| RN-12 | A Course may be added only if it does not already exist in the same School. | RF 8.1 |
| RN-13 | A Course may be deleted only if it has no active associated cohorts, records, or justifications. | RF 8.1 |
| RN-14 | A Cohort must have a valid responsible user before attendance recording begins. | RF 2.3 |
| RN-15 | A Cohort must be associated with a valid Environment. | RF 2.1, RF 2.2 |
| RN-16 | A Responsible User must be a valid system user. | RF 2.3.1 |
| RN-17 | An Environment must exist before being associated with a Cohort or Attendance Record. | RF 2.1 |
| RN-18 | A Schedule cannot overlap another Schedule assigned to the same Responsible User. | RF 2.3.2 |
| RN-19 | A Cohort must have availability for assigning a time block. | RF 2.3.2 |
| RN-20 | A time block must have a start time earlier than its end time. | RF 2.3.2 |

### Attendance Recording

| ID | Rule | Related requirement |
|---|---|---|
| RN-21 | The student must be registered in the system for their attendance to be valid. | RF 3.1 |
| RN-22 | A student must have a single entry record and a single exit record per Schedule. | RF 3.1 |
| RN-23 | The entry or exit record must correspond to the assigned training block. | RF 3.1 |
| RN-24 | Facial recognition is valid only if the facial parameter corresponds to a registered user. | RF 3.1.1 |
