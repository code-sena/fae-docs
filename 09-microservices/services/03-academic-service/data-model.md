# Data Model — Academic Service

> This service is the authoritative owner of academic structure and school operations.
> The academic context maintains the canonical data for the institution's learning model.

---

## Database engine

**Engine:** PostgreSQL
**Justification:** Academic records are relational, strongly indexed, and transactional by design.

---

## Schema

### Enum: Enrollment_Status

| Value |
|-------|
| Active |
| Withdrawn |
| Completed |

### Enum: Days

| Value |
|-------|
| Lunes |
| Martes |
| Miercoles |
| Jueves |
| Viernes |
| Sabado |
| Domingo |

### Table: Student

| Field | Type | Nullable | Description | Constraints |
|-------|------|----------|-------------|-------------|
| student_id | UUID | No | Student identifier | PK |
| school_id | UUID | No | Owning school | Not null |
| person_id | UUID | No | Linked person from Identity context | Unique, external ref |
| student_code | VARCHAR | No | Student code within the school | Not null |
| status | BOOLEAN | No | Student active state | Default true |
| created_at | DATETIME | No | Creation date | Not null |
| updated_at | DATETIME | Yes | Last update | Nullable |

**Indexes:**
- `uq_student_code_school` on `(school_id, student_code)` unique
- `uq_student_person` on `(person_id)` unique

### Table: Teacher

| Field | Type | Nullable | Description | Constraints |
|-------|------|----------|-------------|-------------|
| teacher_id | UUID | No | Teacher identifier | PK |
| school_id | UUID | No | Owning school | Not null |
| person_id | UUID | No | Linked person from Identity context | Unique, external ref |
| teacher_code | VARCHAR | No | Teacher code within school | Not null |
| status | BOOLEAN | No | Teacher active state | Default true |
| created_at | DATETIME | No | Creation date | Not null |
| updated_at | DATETIME | Yes | Last update | Nullable |

**Indexes:**
- `uq_teacher_code_school` on `(school_id, teacher_code)` unique
- `uq_teacher_person` on `(person_id)` unique

### Table: Course

| Field | Type | Nullable | Description | Constraints |
|-------|------|----------|-------------|-------------|
| course_id | INT | No | Course identifier | PK, auto-increment |
| school_id | UUID | No | Owning school | Not null |
| course_name | VARCHAR | No | Course name | Not null |
| course_code | VARCHAR | No | Course code | Not null |
| status | BOOLEAN | No | Course activity status | Default true |
| created_at | DATETIME | No | Creation date | Not null |
| updated_at | DATETIME | Yes | Last update | Nullable |

**Indexes:**
- `uq_course_code_school` on `(school_id, course_code)` unique

### Table: Classroom

| Field | Type | Nullable | Description | Constraints |
|-------|------|----------|-------------|-------------|
| classroom_id | INT | No | Classroom identifier | PK, auto-increment |
| school_id | UUID | No | Owning school | Not null |
| classroom_name | VARCHAR | No | Classroom name or room | Not null |
| status | BOOLEAN | No | Classroom active state | Default true |
| created_at | DATETIME | No | Creation date | Not null |
| updated_at | DATETIME | Yes | Last update | Nullable |

**Indexes:**
- `uq_classroom_name_school` on `(school_id, classroom_name)` unique

### Table: Period

| Field | Type | Nullable | Description | Constraints |
|-------|------|----------|-------------|-------------|
| period_id | INT | No | Period identifier | PK, auto-increment |
| school_id | UUID | No | Owning school | Not null |
| name | VARCHAR | No | Period name | Not null |
| start_date | DATE | No | Start date | Not null |
| end_date | DATE | No | End date | Not null |
| is_active | BOOLEAN | No | Indicates active term | Default false |
| created_at | DATETIME | No | Creation date | Not null |
| updated_at | DATETIME | Yes | Last update | Nullable |

**Indexes:**
- `uq_period_name_school` on `(school_id, name)` unique

### Table: Enrollment

| Field | Type | Nullable | Description | Constraints |
|-------|------|----------|-------------|-------------|
| enrollment_id | BIGINT | No | Enrollment identifier | PK, auto-increment |
| school_id | UUID | No | Owning school | Not null |
| student_id | UUID | No | Student enrolled | FK to `Student.student_id` |
| course_id | INT | No | Course enrolled | FK to `Course.course_id` |
| period_id | INT | No | Academic period | FK to `Period.period_id` |
| enrollment_date | DATETIME | No | Enrollment date | Not null |
| status | Enrollment_Status | No | Enrollment status | Default Active |
| created_at | DATETIME | No | Creation date | Not null |
| updated_at | DATETIME | Yes | Last update | Nullable |

**Indexes:**
- `uq_enrollment_student_course_period` on `(student_id, course_id, period_id)` unique
- `idx_enrollment_student` on `(student_id)`
- `idx_enrollment_course` on `(course_id)`
- `idx_enrollment_period` on `(period_id)`

### Table: Schedule

| Field | Type | Nullable | Description | Constraints |
|-------|------|----------|-------------|-------------|
| schedule_id | BIGINT | No | Schedule identifier | PK, auto-increment |
| school_id | UUID | No | Owning school | Not null |
| period_id | INT | No | Period for the schedule | FK to `Period.period_id` |
| course_id | INT | No | Course in the schedule | FK to `Course.course_id` |
| teacher_id | UUID | No | Assigned teacher | FK to `Teacher.teacher_id` |
| classroom_id | INT | No | Assigned classroom | FK to `Classroom.classroom_id` |
| day | Days | No | Day of the week | Not null |
| start_time | TIME | No | Class start time | Not null |
| end_time | TIME | No | Class end time | Not null |
| created_at | DATETIME | No | Creation date | Not null |
| updated_at | DATETIME | Yes | Last update | Nullable |

**Indexes:**
- `uq_schedule_classroom_slot` on `(classroom_id, day, start_time, period_id)` unique
- `uq_schedule_teacher_slot` on `(teacher_id, day, start_time, period_id)` unique
- `idx_schedule_period` on `(period_id)`
- `idx_schedule_course` on `(course_id)`
- `idx_schedule_teacher` on `(teacher_id)`
- `idx_schedule_classroom` on `(classroom_id)`

---

## Relationships

- `Enrollment.student_id` → `Student.student_id`
- `Enrollment.course_id` → `Course.course_id`
- `Enrollment.period_id` → `Period.period_id`
- `Schedule.period_id` → `Period.period_id`
- `Schedule.course_id` → `Course.course_id`
- `Schedule.teacher_id` → `Teacher.teacher_id`
- `Schedule.classroom_id` → `Classroom.classroom_id`

> `Student.person_id` and `Teacher.person_id` are external references to the Identity context by public ID.

---

## Modeling decisions

- **Academic ownership:** This bounded context owns `Student`, `Teacher`, `Course`, `Classroom`, `Period`, `Enrollment`, and `Schedule`.
- **Person independence:** `Person` is a separate identity aggregate; academic profiles are separate local entities.
- **Slot integrity:** schedule rules prevent overlapping classroom and teacher time assignments.

---

## Schema migration

**Tool:** Flyway
**Script location:** src/migrations/
**Naming convention:** V[NNN]__[description].sql

**Rollback policy:**
- Prefer additive migrations for new courses or periods.
- Validate schedule compatibility before altering constraints.

