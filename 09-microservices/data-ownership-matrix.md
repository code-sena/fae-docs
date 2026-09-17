# Data Ownership Matrix

> Document who is the source of truth for each business entity across the platform. No bounded context should own or directly modify the canonical rows of another context.

---

## 1. Ownership table

| Domain entity | Source of truth | Bounded context | Access pattern | DB |
|---------------|-----------------|-----------------|----------------|-----|
| School | Identity Service | Identity | API + events | PostgreSQL |
| Person | Identity Service | Identity | API + events | PostgreSQL |
| User | Identity Service | Identity | API + events | PostgreSQL |
| User_Session | Identity Service | Identity | API + events | PostgreSQL |
| Role | Authorization Service | Authorization | API + events | PostgreSQL |
| Permission | Authorization Service | Authorization | API + events | PostgreSQL |
| Role_Permission | Authorization Service | Authorization | API + events | PostgreSQL |
| User_Role | Authorization Service | Authorization | API + events | PostgreSQL |
| Student | Academic Service | Academic | API + events | PostgreSQL |
| Teacher | Academic Service | Academic | API + events | PostgreSQL |
| Course | Academic Service | Academic | API + events | PostgreSQL |
| Classroom | Academic Service | Academic | API + events | PostgreSQL |
| Period | Academic Service | Academic | API + events | PostgreSQL |
| Enrollment | Academic Service | Academic | API + events | PostgreSQL |
| Environment | Scheduling Service | Scheduling | API + events | PostgreSQL |
| Schedule_Block | Scheduling Service | Scheduling | API + events | PostgreSQL |
| Class_Session | Scheduling Service | Scheduling | API + events | PostgreSQL |
| Attendance_Record | Attendance Service | Attendance | API + events | PostgreSQL |
| Justification_Type | Attendance Service | Attendance | API + events | PostgreSQL |
| Justification | Attendance Service | Attendance | API + events | PostgreSQL |
| Supporting_Document | Attendance Service | Attendance | API + events | PostgreSQL |
| Facial_Embedding | Biometric Service | Biometric | API + events | MongoDB |
| Fingerprint_Embedding | Biometric Service | Biometric | API + events | MongoDB |
| Facial_Update_Case | Biometric Service | Biometric | API + events | MongoDB |
| Alert_Type | Notification Service | Notification | API + events | PostgreSQL |
| Alert | Notification Service | Notification | API + events | PostgreSQL |
| Academic_Configuration | Configuration Service | Configuration | API + events | PostgreSQL |
| Security_Configuration | Configuration Service | Configuration | API + events | PostgreSQL |
| Password_Policy | Configuration Service | Configuration | API + events | PostgreSQL |

---

## 2. Access patterns explained

| Pattern | Description | When to use |
|---------|-------------|-------------|
| API + events | The owning service exposes both REST endpoints and publishes domain events | Default for all services |
| API only | The service exposes REST endpoints but does not publish events for this entity | Rare; used when no downstream consumer exists |
| Events only | The service publishes events but does not expose direct API access | Used for internal system events (e.g., heartbeats) |

---

## 3. Read access by consumer

| Consumer service | Reads from (via API or events) | Entities consumed |
|------------------|-------------------------------|-------------------|
| API Gateway | All services | Routes requests to owning services |
| Authorization Service | Identity Service | User and school data for role assignment |
| Academic Service | Identity Service | Person data for student/teacher profiles |
| Scheduling Service | Academic Service | Cohort, course, instructor data for schedule blocks |
| Attendance Service | Academic Service | Schedule, enrollment, and classroom data |
| Attendance Service | Scheduling Service | Class session context for attendance recording |
| Attendance Service | Biometric Service | Biometric verification status |
| Biometric Service | Identity Service | User metadata for correlation |
| Notification Service | Academic Service | Academic actor data for alert generation |
| Configuration Service | Identity Service | School data for config initialization |

---

## 4. Rules

1. **No cross-context writes.** A service must never INSERT, UPDATE, or DELETE rows in another service's database.
2. **Projections are local.** When a consumer needs cross-context data, it stores a local projection keyed by the public UUID.
3. **Projections are eventual.** Local projections may lag behind the source of truth; consumers must tolerate stale reads.
4. **Public IDs only.** Cross-context references use UUIDs, never auto-increment local keys.
5. **One writer.** Each canonical table has exactly one service that writes to it.

---

## Correlations

- Service catalog → `09-microservices/service-catalog.md`
- Boundary rules → `09-microservices/service-boundary-rules.md`
- Data model per service → `09-microservices/services/NN-name/data-model.md`
