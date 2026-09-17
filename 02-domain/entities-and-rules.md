# Entities, Value Objects, and Business Rules

> This document defines the main entities in the FaceAttend EDU domain, their value objects, and the business rules that the system must guarantee at all times.
> The goal is for the team to share a common language about business concepts and for the rules to support functional design, testing, and implementation.

> **Stack note:** Code examples (classes, interfaces, decorators) are written in pseudo-TypeScript to illustrate the idea. To see the implementation in your technology:
> [`_stacks/node-typescript.md`](../_stacks/node-typescript.md) ·
> [`_stacks/java-spring.md`](../_stacks/java-spring.md) ·
> [`_stacks/python-fastapi.md`](../_stacks/python-fastapi.md) ·
> [`_stacks/go.md`](../_stacks/go.md)

---

## Conventions

- **Entity:** business concept with its own identity and lifecycle. Example: a Cohort or an Attendance Record.
- **Value object:** concept without its own identity, described by its attributes. Example: a time range or a facial parameter.
- **Invariant:** rule that must always be satisfied within the domain.
- Rules use the `RN` prefix (Business Rule) and relate to SRS requirements.

---

## Entity Map by Context

| Context | Main entities | Value objects |
|---|---|---|
| Identity | Person, App_User, City | Credentials, Session |
| Authorization | Role, Permission, Role_Permission, User_Role | — |
| Academic | School, Program, Academic_Period, Cohort, Course, Academic_Actor_Type, Academic_Actor, Enrollment | — |
| Scheduling | Environment, Schedule_Block, Class_Session | Time Block |
| Attendance | Attendance_Record, Justification_Type, Justification, Supporting_Document | Recording Method |
| Biometric | Facial_Embedding, Fingerprint_Embedding | — |
| Notification | Alert_Type, Alert | — |
| Configuration | Academic_Configuration, Security_Configuration, Biometric_Update_Case | — |

---

## 1. Identity Context

### Person

**Description:** Represents a real individual in the system: student, teacher, or administrator. It is the central entity referenced by all other contexts.

**Identity:** `person_id` (SERIAL).

**Conceptual attributes:**
- First and last names.
- Document number and type.
- Email address.
- Telephone number.
- Birth date.
- Status (active/inactive).
- City of origin.

**Lifecycle:** Registered → Active → Inactive.

**Associated rules:** RN-01, RN-02, RN-03.

### App_User

**Description:** Credentials that allow a Person to access the system. A Person may exist without an active App_User, but an App_User cannot exist without an associated Person.

**Identity:** `user_id` (UUID).

**Conceptual attributes:**
- Username (unique).
- Password hash.
- Status (active/inactive).
- Last access date.
- Associated person.

**Lifecycle:** Created → Active → Inactive.

**Associated rules:** RN-04, RN-05, RN-06.

### City

**Description:** Catalog of cities. Referenced by School in the Academic context.

**Identity:** `city_id` (SERIAL).

**Conceptual attributes:**
- Name.
- Department/state.

**Associated rules:** RN-11.

---

## 2. Authorization Context

### Role

**Description:** Defines an access level within the system. Roles are global (not per-school). The roles planned in the SRS are:
- Administrator.
- Instructor / Teacher.
- Student / Learner.

**Identity:** `role_id` (SERIAL).

**Associated rules:** RN-07, RN-08.

### Permission

**Description:** Specific action that a role may perform in the system, such as registering users, approving justifications, exporting reports, or consulting histories.

**Identity:** `permission_id` (SERIAL).

**Associated rules:** RN-09, RN-10.

### Role_Permission

**Description:** Associates a role with a permission. Defines which actions each role may perform.

**Identity:** Composite (`role_id`, `permission_id`).

### User_Role

**Description:** Assigns a role to a specific app_user. A user may have multiple roles across different schools.

**Identity:** Composite (`user_id`, `role_id`, `school_id`).

---

## 3. Academic Context

### School

**Description:** Educational institution that uses FaceAttend EDU. It groups programs, cohorts, environments, people, and records. Multi-tenant root entity.

**Identity:** `school_id` (SERIAL).

**Conceptual attributes:**
- Code (unique per school).
- Name.
- City (reference to Identity.City).
- Address, phone, email.
- Status (active/inactive).

**Associated rules:** RN-11, RN-12.

### Program

**Description:** Academic program offered by a school (e.g., "Software Development", "Office Automation"). Groups cohorts and courses.

**Identity:** `program_id` (SERIAL).

**Conceptual attributes:**
- Name.
- Description.
- Associated school.
- Status.

**Associated rules:** RN-12.

### Academic_Period

**Description:** Time division within a school (e.g., "2024-Q1", "2024-Semester 1"). Cohorts and enrollments are scoped to a period.

**Identity:** `period_id` (SERIAL).

**Conceptual attributes:**
- Name.
- Start and end dates.
- Associated school.
- Status.

### Cohort

**Description:** Group of students associated with a program, an academic period, and optionally an instructor. It is the main unit for attendance control.

**Identity:** `cohort_id` (SERIAL).

**Conceptual attributes:**
- Name.
- Associated program.
- Associated academic period.
- Associated school.
- Start and end dates.
- Status.

**Lifecycle:** Registered → Active → Finished.

**Associated rules:** RN-14, RN-15.

### Course

**Description:** Subject or module within a program (e.g., "Mathematics I", "Programming Fundamentals").

**Identity:** `course_id` (SERIAL).

**Conceptual attributes:**
- Name.
- Code.
- Description.
- Credit hours.
- Associated program.
- Status.

**Associated rules:** RN-12, RN-13.

### Academic_Actor_Type

**Description:** Catalog of actor types: Student, Instructor.

**Identity:** `actor_type_id` (SERIAL).

**Conceptual attributes:**
- Name (e.g., "Student", "Instructor").
- Description.

### Academic_Actor

**Description:** Unified representation of a person within an academic context. Replaces separate Student/Teacher tables. An academic_actor references a Person via external ID and has an actor_type.

**Identity:** `academic_actor_id` (BIGSERIAL).

**Conceptual attributes:**
- Person reference (external ID → Identity.Person).
- Actor type (FK → Academic_Actor_Type).
- Code (institutional code).
- Status.

**Associated rules:** RN-16, RN-21.

### Enrollment

**Description:** Links an academic_actor (student) to a cohort. Tracks the student's participation in a specific cohort.

**Identity:** `enrollment_id` (SERIAL).

**Conceptual attributes:**
- Academic actor reference.
- Cohort reference.
- Enrollment date.
- Status.

**Associated rules:** RN-14, RN-21.

---

## 4. Scheduling Context

### Environment

**Description:** Physical space where classes are held (classroom, lab, auditorium). Assigned to a school.

**Identity:** `environment_id` (SERIAL).

**Conceptual attributes:**
- Code.
- Name.
- Capacity.
- Associated school.
- Status.

**Associated rules:** RN-15, RN-17.

### Schedule_Block

**Description:** Recurring time slot defining when a cohort meets with an instructor in an environment. Prevents double-booking through unique constraints.

**Identity:** `schedule_block_id` (SERIAL).

**Conceptual attributes:**
- Cohort reference (external ID → Academic.Cohort).
- Course reference (external ID → Academic.Course).
- Instructor actor reference (external ID → Academic.Academic_Actor).
- Environment reference.
- Day of week.
- Start and end times.

**Associated rules:** RN-18, RN-19, RN-20.

### Class_Session

**Description:** Concrete instance of a schedule_block on a specific date. This is the unit to which attendance records are attached.

**Identity:** `class_session_id` (BIGSERIAL).

**Conceptual attributes:**
- Schedule block reference.
- Date.
- Status (Open, Closed, Cancelled).

**Associated rules:** RN-18, RN-22.

---

## 5. Attendance Context

### Attendance_Record

**Description:** Individual attendance record of an academic_actor in a class_session. The central business entity of FaceAttend EDU.

**Identity:** `attendance_record_id` (BIGSERIAL).

**Conceptual attributes:**
- Class session reference (external ID → Scheduling.Class_Session).
- Academic actor reference (external ID → Academic.Academic_Actor).
- Captured at (timestamp).
- Attendance status: Present, Absent, Late, Justified.
- Capture method: FACIAL, MANUAL, IMPORT.
- Match score (biometric confidence).

**Lifecycle:** Recorded → Validated → Justified if applicable.

**Associated rules:** RN-22, RN-23, RN-24, RN-25, RN-26.

### Justification_Type

**Description:** Catalog of justification types for absences (e.g., "Medical leave", "Personal emergency").

**Identity:** `justification_type_id` (SERIAL).

**Conceptual attributes:**
- Name (unique).
- Description.
- Requires attachment (boolean).
- Status.

### Justification

**Description:** Request submitted by a student to explain an absence. Linked 1:1 to an attendance_record.

**Identity:** `justification_id` (BIGSERIAL).

**Conceptual attributes:**
- Attendance record reference (unique).
- Justification type reference.
- Reason (text).
- Submitted at.
- Review status: Pending, Approved, Rejected.
- Reviewed by (external ID → Identity.App_User).
- Reviewed at.

**Lifecycle:** Pending → Approved / Rejected.

**Associated rules:** RN-32, RN-33, RN-34, RN-35, RN-36.

### Supporting_Document

**Description:** File attached to a justification (medical certificates, evidence).

**Identity:** `supporting_document_id` (BIGSERIAL).

**Conceptual attributes:**
- Justification reference.
- File name.
- Storage URI.
- MIME type.
- Size in bytes.

**Associated rules:** RN-33, RN-34.

---

## 6. Biometric Context

> This context persists entirely in **MongoDB** (NoSQL). Entities below represent document collections, not SQL tables.

### Facial_Embedding

**Description:** Numeric vector representing a person's face. Used for facial recognition during attendance capture.

**Identity:** `_id` (ObjectId).

**Conceptual attributes:**
- Person reference (external ID → Identity.Person).
- Encoding (vector of floats).
- Model version.
- Template version (incremented on re-enrollment).
- Enrolled at.
- Active/inactive.

**Associated rules:** RN-27, RN-28, RN-29.

### Fingerprint_Embedding

**Description:** Fingerprint biometric template. Supports up to 10 fingers per person.

**Identity:** `_id` (ObjectId).

**Conceptual attributes:**
- Person reference (external ID → Identity.Person).
- Finger number (1..10).
- Encoding (template vector).
- Model version.
- Template version.
- Enrolled at.
- Active/inactive.

**Associated rules:** RN-27, RN-28.

---

## 7. Notification Context

### Alert_Type

**Description:** Catalog of alert types (e.g., "Absence threshold reached", "Schedule conflict").

**Identity:** `alert_type_id` (SMALLINT).

**Conceptual attributes:**
- Code (unique).
- Name.

### Alert

**Description:** Notification generated by the system for relevant events, linked to an academic_actor.

**Identity:** `alert_id` (BIGSERIAL).

**Conceptual attributes:**
- Academic actor reference (external ID → Academic.Academic_Actor).
- Alert type reference.
- Raised at.
- Resolved at (NULL = unresolved).

**Associated rules:** RN-42, RN-43.

---

## 8. Configuration Context

### Academic_Configuration

**Description:** Configurable parameters per school that affect academic behavior (e.g., `tardy_tolerance_minutes`).

**Identity:** `configuration_id` (SERIAL).

**Conceptual attributes:**
- School reference (external ID → Academic.School).
- Configuration name.
- Configuration value (stored as string).
- Description.

### Security_Configuration

**Description:** System-wide security parameters (not per-school). E.g., `max_login_attempts`, `session_timeout_minutes`.

**Identity:** `configuration_id` (SERIAL).

**Conceptual attributes:**
- Configuration name (unique).
- Configuration value.
- Description.

### Biometric_Update_Case

**Description:** Unified case for requesting facial or fingerprint embedding updates. Tracks the full workflow from request to review.

**Identity:** `case_id` (UUID).

**Conceptual attributes:**
- Person reference (external ID → Identity.Person).
- Biometric type: FACIAL or FINGERPRINT.
- Finger number (only if FINGERPRINT).
- Current embedding reference (logical _id in MongoDB).
- Reason.
- Update status: Pending, In_Review, Approved, Rejected.
- Requested by, requested at.
- Reviewed by, reviewed at.
- Resolution notes.

**Lifecycle:** Pending → In_Review → Approved / Rejected.

---

## Value Objects

### Credentials

**Context:** Identity.

**Description:** Set of data required to authenticate an account (username + password). It has no identity of its own outside the app_user.

**Associated rules:** RN-04, RN-44.

### Session

**Context:** Identity.

**Description:** Temporary state indicating that a user has been authenticated and may interact with the system for a limited period.

**Associated rules:** RN-04, RN-45.

### Time_Block

**Context:** Scheduling.

**Description:** Time interval consisting of a day-of-week, start time, and end time. Used in schedule blocks.

**Associated rules:** RN-18, RN-19.

### Recording_Method

**Context:** Attendance.

**Description:** Indicates how attendance was captured. Values: FACIAL, MANUAL, IMPORT.

**Associated rules:** RN-25, RN-26.

---

## Business Rules

### Identity

| ID | Rule | Related requirement |
|---|---|---|
| RN-01 | Every App_User must be associated with a valid Person. | RF 1.1, RF 1.1.1 |
| RN-02 | A Person may exist without an active App_User, but there cannot be active academic records without an associated Person. | RF 1.6, RF 1.7 |
| RN-03 | A Person cannot be deleted if they have active associated attendance records, enrollments, or justifications. | RF 1.7 |
| RN-04 | Every user must authenticate before accessing system functions. | RF 1.3, RNF 4 |
| RN-05 | Credentials must be stored and transmitted securely. | RNF 4 |
| RN-06 | An administrator may activate or deactivate users without deleting associated historical information. | RF 1.6 |

### Authorization

| ID | Rule | Related requirement |
|---|---|---|
| RN-07 | Every App_User must have at least one assigned role. | RF 1.1.1 |
| RN-08 | A user's role may be changed only by a user with administrative permissions. | RF 4.6 |
| RN-09 | Access permissions must depend on the assigned role. | RF 1.1.2 |
| RN-10 | A user cannot execute an action without permission associated with their role. | RF 1.1.2 |

### Academic

| ID | Rule | Related requirement |
|---|---|---|
| RN-11 | All academic data must belong to a School. | RF 8 |
| RN-12 | A Program may be added only if it does not already exist in the same School. | RF 8.1 |
| RN-13 | A Course may be deleted only if it has no active associated enrollments or attendance records. | RF 8.1 |
| RN-14 | A Cohort must have a valid instructor before attendance recording begins. | RF 2.3 |
| RN-15 | A Cohort must be associated with a valid Environment. | RF 2.1, RF 2.2 |
| RN-16 | An Academic_Actor must reference a valid Person. | RF 2.3.1 |

### Scheduling

| ID | Rule | Related requirement |
|---|---|---|
| RN-17 | An Environment must exist before being associated with a Cohort or Class_Session. | RF 2.1 |
| RN-18 | A Schedule_Block cannot overlap another Schedule_Block assigned to the same Instructor in the same Environment. | RF 2.3.2 |
| RN-19 | A Cohort must have availability for assigning a time block. | RF 2.3.2 |
| RN-20 | A time block must have a start time earlier than its end time. | RF 2.3.2 |

### Attendance

| ID | Rule | Related requirement |
|---|---|---|
| RN-21 | The student must be an active Academic_Actor for their attendance to be valid. | RF 3.1 |
| RN-22 | An academic_actor can have only one attendance_record per class_session (uniqueness constraint). | RF 3.1 |
| RN-23 | The attendance record must correspond to an Open class_session. | RF 3.1 |
| RN-24 | Facial recognition is valid only if the biometric embedding corresponds to a registered person. | RF 3.1.1 |
| RN-25 | The capture_method must be one of: FACIAL, MANUAL, IMPORT. | RF 3.1, RF 3.1.2 |
| RN-26 | If facial scanning fails, a MANUAL or IMPORT record may be used. | RF 3.1.2 |

### Biometric

| ID | Rule | Related requirement |
|---|---|---|
| RN-27 | Biometric embeddings (facial or fingerprint) reference a Person via external ID — no cross-context FK. | RNF 4 |
| RN-28 | Each person may have only one active facial embedding at a time (is_active flag). | RF 3.1.1 |
| RN-29 | Facial embedding updates require a Biometric_Update_Case approved by an administrator. | RF 3.1.1 |

### Justification

| ID | Rule | Related requirement |
|---|---|---|
| RN-32 | A justification must reference exactly one attendance_record (1:1 relationship). | RF 6 |
| RN-33 | If the justification_type requires an attachment, at least one supporting_document must be uploaded. | RF 6.1 |
| RN-34 | Supporting documents must be valid according to criteria defined by the administrator. | RF 6.1 |
| RN-35 | A justification must be reviewed by a user with appropriate permissions. | RF 6.2 |
| RN-36 | When a justification is approved, the attendance_status updates to "Justified". | RF 6.2 |

### Notification

| ID | Rule | Related requirement |
|---|---|---|
| RN-42 | Alerts are generated by the system for events such as absence thresholds or schedule conflicts. | RF 4.1 |
| RN-43 | An alert may be resolved by a user with appropriate permissions. | RF 4.1 |

---

## Correlations

- Domain entities that map to these tables → `06-data/domains/`
- Saga and Outbox pattern for distributed consistency → `05-architecture/pattern-guide.md`
- Data model conventions → `06-data/models.md`
- How data is accessed via API → `07-api/contracts/openapi/`
