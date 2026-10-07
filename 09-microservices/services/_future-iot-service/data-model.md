# Data Model — IoT Service

> This service is the authoritative owner of the device lifecycle and classroom assignment metadata.
> It manages hardware identity and current assignment state without owning academic business rules.

---

## Database engine

**Engine:** PostgreSQL
**Justification:** Device assignments are relational and require historical tracking of location changes, status, and maintenance periods.

---

## Schema

### Enum: Device_Status

| Value |
|-------|
| Active |
| Inactive |
| Maintenance |

### Table: IoT_Device

| Field | Type | Nullable | Description | Constraints |
|-------|------|----------|-------------|-------------|
| device_id | UUID | No | Device identifier | PK |
| school_id | UUID | No | Owning school | Not null |
| device_name | VARCHAR | No | Device name | Not null |
| mac_address | VARCHAR | Yes | MAC address | Unique |
| ip_address | VARCHAR | Yes | IP or network address | Optional |
| status | Device_Status | No | Device state | Default Active |
| last_connection | DATETIME | Yes | Last successful communication | Nullable |
| observation | TEXT | Yes | Free text notes | Optional |
| created_at | DATETIME | No | Creation time | Not null |
| updated_at | DATETIME | Yes | Last update | Nullable |

**Indexes:**
- `uq_device_mac` on `(mac_address)` unique
- `idx_device_school` on `(school_id)`
- `idx_device_status` on `(status)`

### Table: Device_Assignment

| Field | Type | Nullable | Description | Constraints |
|-------|------|----------|-------------|-------------|
| assignment_id | BIGINT | No | Assignment record identifier | PK, auto-increment |
| device_id | UUID | No | Device assigned | FK to `IoT_Device.device_id` |
| classroom_id | INT | No | Classroom where the device is assigned | External reference |
| assigned_from | DATETIME | No | Assignment start | Not null |
| assigned_to | DATETIME | Yes | Assignment end | Nullable |
| status | BOOLEAN | No | Whether the assignment is active | Default true |
| created_at | DATETIME | No | Creation time | Not null |

**Indexes:**
- `idx_assignment_device` on `(device_id)`
- `idx_assignment_classroom` on `(classroom_id)`
- `uq_device_assignment_start` on `(device_id, assigned_from)` unique

---

## Relationships

- `Device_Assignment.device_id` → `IoT_Device.device_id`

> `classroom_id` is kept as a public reference to the Academic bounded context; no cross-context FK is enforced.

---

## Modeling decisions

- **Independent hardware ownership:** the IoT service tracks devices and their assignments without owning the classroom schema.
- **Historical provenance:** device assignments are kept as time-bounded records so reallocation can be audited.
- **Operational resilience:** the service supports status changes, network checks, and maintenance without coupling to academic rules.

---

## Schema migration

**Tool:** Liquibase 4.29 (chained containers, ADR-010)
**Script location:** src/migrations/
**Naming convention:** V[NNN]__[description].sql

**Rollback policy:**
- Preserve historical assignment data when reassigning devices.
- Prefer additive migration for new statuses or operational metadata.

