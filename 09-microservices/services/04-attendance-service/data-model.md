# Data Model — Attendance Service

> This service is the authoritative owner of class sessions and attendance outcomes.
> It captures the real attendance events during classes and the associated justification flow.

---

## Database engine

**Engine:** PostgreSQL
**Justification:** Attendance state is transactional and needs strong consistency for daily operations and reporting.

---

## Schema

### Enum: Attendance_Status

| Value |
|-------|
| Present |
| Absent |
| Late |
| Justified |

### Enum: Class_Session_Status

| Value |
|-------|
| Open |
| Closed |
| Cancelled |

### Enum: Approval_Status

| Value |
|-------|
| Pending |
| Approved |
| Rejected |

### Table: Class_Session

| Field | Type | Nullable | Description | Constraints |
|-------|------|----------|-------------|-------------|
| session_id | BIGINT | No | Session identifier | PK, auto-increment |
| school_id | UUID | No | Owning school | Not null |
| schedule_id | BIGINT | No | Related scheduled class | External reference |
| date | DATE | No | Session date | Not null |
| opened_by | UUID | Yes | Teacher who opened the session | External ID |
| teacher_entry_time | TIME | Yes | Opening time | Optional |
| closed_by | UUID | Yes | Teacher who closed the session | External ID |
| teacher_exit_time | TIME | Yes | Closing time | Optional |
| session_status | Class_Session_Status | No | Session lifecycle state | Default Open |
| created_at | DATETIME | No | Creation time | Not null |
| updated_at | DATETIME | Yes | Last update | Nullable |

**Indexes:**
- `uq_session_schedule_date` on `(schedule_id, date)` unique
- `idx_session_schedule` on `(schedule_id)`
- `idx_session_date` on `(date)`

### Table: Attendance

| Field | Type | Nullable | Description | Constraints |
|-------|------|----------|-------------|-------------|
| attendance_id | BIGINT | No | Attendance record identifier | PK, auto-increment |
| school_id | UUID | No | Owning school | Not null |
| student_id | UUID | No | Student involved | External reference |
| session_id | BIGINT | No | Class session attended | FK to `Class_Session.session_id` |
| device_id | UUID | Yes | Capture device if present | Optional external ID |
| time | TIME | No | Entry time | Not null |
| exit_time | TIME | Yes | Exit time | Optional |
| status | Attendance_Status | No | Attendance result | Not null |
| source | VARCHAR | No | Source of record | Manual, IoT, Biometric, Import |
| created_at | DATETIME | No | Creation time | Not null |
| updated_at | DATETIME | Yes | Last update | Nullable |

**Indexes:**
- `uq_attendance_student_session` on `(student_id, session_id)` unique
- `idx_attendance_student` on `(student_id)`
- `idx_attendance_session` on `(session_id)`
- `idx_attendance_device` on `(device_id)`

### Table: Justification_Type

| Field | Type | Nullable | Description | Constraints |
|-------|------|----------|-------------|-------------|
| justification_type_id | INT | No | Justification type identifier | PK, auto-increment |
| school_id | UUID | No | Owning school | Not null |
| name | VARCHAR | No | Example: medical leave, family emergency | Not null |
| description | VARCHAR | Yes | Explanation of the type | Optional |
| requires_attachment | BOOLEAN | No | Requires supporting evidence | Default false |
| status | BOOLEAN | No | Active or inactive type | Default true |
| created_at | DATETIME | No | Creation time | Not null |
| updated_at | DATETIME | Yes | Last update | Nullable |

**Indexes:**
- `uq_justification_type_name_school` on `(school_id, name)` unique

### Table: Justification

| Field | Type | Nullable | Description | Constraints |
|-------|------|----------|-------------|-------------|
| justification_id | BIGINT | No | Justification identifier | PK, auto-increment |
| attendance_id | BIGINT | No | Related attendance | FK to `Attendance.attendance_id`, unique |
| justification_type_id | INT | No | Type of justification | FK to `Justification_Type.justification_type_id` |
| description | TEXT | No | Reason associated | Not null |
| justification_attachment | VARCHAR | Yes | File path or URL | Optional |
| approval | Approval_Status | No | Review decision | Default Pending |
| created_at | DATETIME | No | Creation date | Not null |
| reviewed_by | UUID | Yes | Reviewer ID from Identity context | External ref |
| reviewed_at | DATETIME | Yes | Decision date | Nullable |

**Indexes:**
- `uq_justification_attendance` on `(attendance_id)` unique
- `idx_justification_type` on `(justification_type_id)`
- `idx_justification_reviewer` on `(reviewed_by)`

---

## Relationships

- `Attendance.session_id` → `Class_Session.session_id`
- `Justification.attendance_id` → `Attendance.attendance_id`
- `Justification.justification_type_id` → `Justification_Type.justification_type_id`

> `student_id`, `schedule_id`, `opened_by`, `closed_by`, `device_id`, and `reviewed_by` are public external IDs or service references, not local FKs.

---

## Modeling decisions

- **Explicit session model:** class state is represented by `Class_Session`, not by repeating schedule-date data on every attendance row.
- **Attendance safety:** attendance validation is bound to an open class session.
- **Decoupling from Identity:** cross-context user references are resolved externally, which keeps the attendance service independent of user deletion or identity service changes.

---

## Schema migration

**Tool:** Flyway
**Script location:** src/migrations/
**Naming convention:** V[NNN]__[description].sql

**Rollback policy:**
- Preserve historical attendance rows; avoid deleting or rewriting attendance records.
- Prefer additive updates to justification and session states.

