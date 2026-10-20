# Data Ownership Matrix

> Document who is the source of truth for each business entity across the platform. No bounded context should own or directly modify the canonical rows of another context.

---

## 1. Ownership table

| Domain entity | Source of truth | Bounded context | Access pattern | DB |
|---------------|-----------------|-----------------|----------------|-----|
| Person | Identity Service | Identity | API + events | PostgreSQL |
| App_User | Identity Service | Identity | API + events | PostgreSQL |
| User_Session | Identity Service | Identity | API + events | PostgreSQL |
| Password_Policy | Identity Service | Identity | API + events | PostgreSQL |
| Role | Authorization Service | Authorization | API + events | PostgreSQL |
| Permission | Authorization Service | Authorization | API + events | PostgreSQL |
| Role_Permission | Authorization Service | Authorization | API + events | PostgreSQL |
| User_Role | Authorization Service | Authorization | API + events | PostgreSQL |
| School | Academic Service | Academic | API + events | PostgreSQL |
| Program | Academic Service | Academic | API + events | PostgreSQL |
| Academic_Period | Academic Service | Academic | API + events | PostgreSQL |
| Cohort | Academic Service | Academic | API + events | PostgreSQL |
| Course | Academic Service | Academic | API + events | PostgreSQL |
| Academic_Actor_Type | Academic Service | Academic | API + events | PostgreSQL |
| Academic_Actor | Academic Service | Academic | API + events | PostgreSQL |
| Enrollment | Academic Service | Academic | API + events | PostgreSQL |
| Environment | Scheduling Service | Scheduling | API + events | PostgreSQL |
| Schedule_Block | Scheduling Service | Scheduling | API + events | PostgreSQL |
| Class_Session | Scheduling Service | Scheduling | API + events | PostgreSQL |
| Attendance_Record | Attendance Service | Attendance | API + events | PostgreSQL |
| Justification_Type | Attendance Service | Attendance | API + events | PostgreSQL |
| Justification | Attendance Service | Attendance | API + events | PostgreSQL |
| Supporting_Document | Attendance Service | Attendance | API + events | PostgreSQL |
| Attendance_Report | Attendance Service | Attendance (derived projection, no FKs) | API | PostgreSQL |
| Facial_Embedding | Biometric Service | Biometric | API + events | MongoDB |
| Fingerprint_Embedding | Biometric Service | Biometric | API + events | MongoDB |
| Alert_Type | Notification Service | Notification | API + events | PostgreSQL |
| Alert | Notification Service | Notification | API + events | PostgreSQL |
| Academic_Configuration | Configuration Service | Configuration | API + events | PostgreSQL |
| Security_Configuration | Configuration Service | Configuration | API + events | PostgreSQL |
| Biometric_Update_Case | Configuration Service (physical host; **logical owner = Biometric**) | Configuration | API + events | PostgreSQL |

> **Removed contexts/entities:** `City` (dropped by Liquibase; cities come from the external
> `countriesnow.space` API) and the **Audit Service** (`audit_log`/`error_log`) — the Audit bounded
> context was eliminated from the model; observability is platform-provided. The 7 `quality` tables
> (`quality_project`, `quality_evaluation`, `quality_evaluation_item`, `process_assessment`,
> `process_assessment_rating`, `istqb_assessment`, `istqb_assessment_item`) are quality tooling
> owned by `09-ms-quality`, not a bounded context.

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
| Authorization Service | Identity Service | App_User and Person data for role assignment |
| Academic Service | Identity Service | Person data for Academic_Actor |
| Scheduling Service | Academic Service | Cohort, Course, Academic_Actor, School data for Schedule_Block/Environment |
| Attendance Service | Scheduling Service | Class_Session context for Attendance_Record |
| Attendance Service | Academic Service | Academic_Actor data for attendance validation |
| Attendance Service | Biometric Service | Embedding verification status (events) |
| Biometric Service | Identity Service | Person metadata for correlation |
| Notification Service | Academic Service | Academic_Actor data for alert generation |
| Configuration Service | Identity Service | Person/App_User for Biometric_Update_Case workflow |
| Configuration Service | Academic Service | School data for Academic_Configuration |

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
