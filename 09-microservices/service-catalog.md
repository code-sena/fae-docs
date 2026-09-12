# Microservices Catalog

> This catalog reflects the current bounded-context model: each service owns a business domain and the data within it, while cross-context references remain public IDs and event-driven integration.

---

## Service map

```
                     ┌──────────────────────┐
  External clients    │ API Gateway / Edge   │
  (Web, Mobile, API) ─▶│ Routing, auth, rate │
                     │ limiting, orchestration │
                     └──────────┬───────────┘
                                │
                     ┌──────────▼───────────┐
                     │ Domain services      │
                     │ 01 Identity          │
                     │ 02 Authorization     │
                     │ 03 Academic          │
                     │ 04 Attendance        │
                     │ 05 Biometric         │
                     │ 06 IoT               │
                     │ 07 Audit             │
                     │ 08 Configuration     │
                     └──────────┬───────────┘
                                │
                     ┌──────────▼───────────┐
                     │ Event transport      │
                     │ Kafka / RabbitMQ     │
                     └──────────────────────┘
```

> The API gateway is infrastructure. The business model itself is formed by the eight bounded contexts above.

---

## Service registry

| # | Service | Main responsibility | Port | DB | DB type | Status |
|---|---------|-------------------|------|-----|---------|--------|
| 01 | `identity-service` | School, person, user, and authenticated session records | 8081 | PostgreSQL | Relational | 🟢 Active |
| 02 | `auth-service` | User authentication, login, registration, and JWT token management | 8082 | Redis | Cache | 🟢 Active |
| 03 | `authorization-service` | Roles, permissions, and role assignment policies | 8083 | PostgreSQL | Relational | 🟢 Active |
| 04 | `academic-service` | Students, teachers, courses, periods, schedules, and enrollments | 8084 | PostgreSQL | Relational | 🟢 Active |
| 05 | `attendance-service` | Real class sessions, attendance, and justification processing | 8085 | PostgreSQL | Relational | 🟢 Active |
| 06 | `biometric-service` | Facial and fingerprint embeddings and update review cases | 8086 | MongoDB | NoSQL / Document | 🟢 Active |
| 07 | `iot-service` | Physical devices and classroom assignment lifecycle | 8087 | PostgreSQL | Relational | 🟢 Active |
| 08 | `audit-service` | Traceability, operational audit, and error records | 8088 | PostgreSQL | Relational | 🟢 Active |
| 09 | `configuration-service` | Academic and security configuration values and password policy | 8089 | PostgreSQL | Relational | 🟢 Active |
| 99 | `api-gateway` | Edge routing and security enforcement for external clients | 8080 | Redis | Cache | 🟢 Active |

**Status legend:** 🟢 Active | 🟡 In development | 🔴 Planned | ⏸ Deprecated

---

## Detail per service

### 01 — `identity-service`

| Field | Value |
|-------|-------|
| **Folder** | `09-microservices/services/01-identity-service/` |
| **Responsibility** | Owns school master data, person profiles, user credentials, and authentication sessions |
| **Type** | Core domain service |
| **Port** | 8081 |
| **DB** | PostgreSQL |
| **Bounded context** | Identity |
| **Externally exposed** | Yes, via API Gateway |

**Owned entities:**
- `School`
- `Person`
- `User`
- `User_Session`

**Cross-context rule:**
- `Person.school_id` and `User.person_id` remain internal to Identity.
- Authorization and other services use public IDs only.

---

### 02 — `auth-service`

| Field | Value |
|-------|-------|
| **Folder** | `09-microservices/services/02-auth-service/` |
| **Responsibility** | Handles user authentication, login, registration, and JWT token lifecycle |
| **Type** | Infrastructure domain service |
| **Port** | 8082 |
| **DB** | Redis (session/cache) |
| **Bounded context** | Authentication (cross-cutting) |
| **Externally exposed** | Yes, via API Gateway |

**Owned entities:**
- User authentication sessions (Redis-backed)
- JWT access and refresh tokens

**Cross-context rule:**
- Does not own user credentials or person data — delegates identity verification to
  `identity-service`.
- Provides authentication tokens consumed by the API Gateway and other services.

---

### 03 — `authorization-service`

| Field | Value |
|-------|-------|
| **Folder** | `09-microservices/services/02-authorization-service/` |
| **Responsibility** | Defines roles, permissions, and assignments that authorize access across the platform |
| **Type** | Core domain service |
| **Port** | 8083 |
| **DB** | PostgreSQL |
| **Bounded context** | Authorization |
| **Externally exposed** | Yes, via API Gateway |

**Owned entities:**
- `Role`
- `Permission`
- `Role_Permission`
- `User_Role`

**Cross-context rule:**
- `User_Role.user_id` references the public `Identity.User.user_id` without a direct FK.

---

### 04 — `academic-service`

| Field | Value |
|-------|-------|
| **Folder** | `09-microservices/services/03-academic-service/` |
| **Responsibility** | Manages students, teachers, courses, periods, classrooms, enrollments, and the timetable |
| **Type** | Core domain service |
| **Port** | 8084 |
| **DB** | PostgreSQL |
| **Bounded context** | Academic |
| **Externally exposed** | Yes, via API Gateway |

**Owned entities:**
- `Student`
- `Teacher`
- `Course`
- `Classroom`
- `Period`
- `Enrollment`
- `Schedule`

**Cross-context rule:**
- `Student.person_id` and `Teacher.person_id` are public IDs, not local FKs.

---

### 05 — `attendance-service`

| Field | Value |
|-------|-------|
| **Folder** | `09-microservices/services/04-attendance-service/` |
| **Responsibility** | Records actual class sessions, individual attendance data, and justification handling |
| **Type** | Core domain service |
| **Port** | 8085 |
| **DB** | PostgreSQL |
| **Bounded context** | Attendance |
| **Externally exposed** | Yes, via API Gateway |

**Owned entities:**
- `Class_Session`
- `Attendance`
- `Justification_Type`
- `Justification`

**Cross-context rule:**
- `student_id`, `schedule_id`, and `opened_by` are public references without FKs to other contexts.

---

### 06 — `biometric-service`

| Field | Value |
|-------|-------|
| **Folder** | `09-microservices/services/05-biometric-service/` |
| **Responsibility** | Stores biometric templates and review cases for recognition workflows |
| **Type** | Specialized domain service |
| **Port** | 8086 |
| **DB** | MongoDB (document store with vector search indexing) |
| **Bounded context** | Biometric |
| **Externally exposed** | Yes, via API Gateway |

**Owned entities:**
- `Facial_Embedding`
- `Fingerprint_Embedding`
- `Facial_Update_Case`

**Cross-context rule:**
- `subject_id` is a public identity reference, not a foreign key to the Identity database.

**Database note:** Per ADR-003, Biometric uses MongoDB instead of PostgreSQL. This is a
documented exception to the single-database strategy in ADR-002, justified by the specialized
vector search requirements of biometric embedding data.

---

### 07 — `iot-service`

| Field | Value |
|-------|-------|
| **Folder** | `09-microservices/services/06-iot-service/` |
| **Responsibility** | Tracks physical devices and their assignment history across classrooms |
| **Type** | Support domain service |
| **Port** | 8087 |
| **DB** | PostgreSQL |
| **Bounded context** | IoT |
| **Externally exposed** | Yes, via API Gateway |

**Owned entities:**
- `IoT_Device`
- `Device_Assignment`

**Cross-context rule:**
- `classroom_id` is a public reference from Academic and remains outside a direct FK.

---

### 08 — `audit-service`

| Field | Value |
|-------|-------|
| **Folder** | `09-microservices/services/07-audit-service/` |
| **Responsibility** | Records business and technical actions for traceability, governance, and incident review |
| **Type** | Support domain service |
| **Port** | 8088 |
| **DB** | PostgreSQL |
| **Bounded context** | Audit |
| **Externally exposed** | Yes, but primarily for internal and operational read access |

**Owned entities:**
- `Audit_Event`
- `Error_Log`

**Cross-context rule:**
- `actor_id` and `user_id` are external identifiers stored as plain IDs rather than FK constraints.

---

### 09 — `configuration-service`

| Field | Value |
|-------|-------|
| **Folder** | `09-microservices/services/08-configuration-service/` |
| **Responsibility** | Holds academic, security, and password-related configuration values used by the platform |
| **Type** | Support domain service |
| **Port** | 8089 |
| **DB** | PostgreSQL |
| **Bounded context** | Configuration |
| **Externally exposed** | Yes, via API Gateway |

**Owned entities:**
- `Academic_Configuration`
- `Security_Configuration`
- `Password_Policy`

**Cross-context rule:**
- Configuration values are operational metadata and do not own the core business records of other services.

---

### 99 — `api-gateway`

| Field | Value |
|-------|-------|
| **Folder** | `09-microservices/services/01-api-gateway/` |
| **Responsibility** | Routes requests, enforces authentication and policy boundaries, and exposes the platform to clients |
| **Type** | Infrastructure service |
| **Port** | 8080 |
| **DB** | Redis |
| **Bounded context** | None; cross-cutting infrastructure |
| **Externally exposed** | Yes |

**Notes:**
- The gateway is not a business owner in the bounded-context model.
- It coordinates calls among the domain services and centralizes external access.

---

## Communication matrix

| Source | Target | Channel | Pattern |
|--------|--------|---------|--------|
| API Gateway | Identity Service | HTTP/REST | Synchronous |
| API Gateway | Authorization Service | HTTP/REST | Synchronous |
| API Gateway | Academic Service | HTTP/REST | Synchronous |
| API Gateway | Attendance Service | HTTP/REST | Synchronous |
| API Gateway | Biometric Service | HTTP/REST | Synchronous |
| API Gateway | IoT Service | HTTP/REST | Synchronous |
| API Gateway | Audit Service | HTTP/REST | Synchronous |
| API Gateway | Configuration Service | HTTP/REST | Synchronous |
| Identity Service | Authorization Service | Event | Asynchronous |
| Academic Service | Attendance Service | Event | Asynchronous |
| Attendance Service | Audit Service | Event | Asynchronous |
| Biometric Service | Attendance Service | Event | Asynchronous |
| IoT Service | Attendance Service | Event | Asynchronous |
| Configuration Service | Academic Service | Event | Asynchronous |

---

## Data ownership matrix

| Domain entity | Source of truth | Access pattern |
|---------------|-----------------|----------------|
| School, Person, User, User_Session | Identity Service | API + events |
| Role, Permission, User_Role | Authorization Service | API + events |
| Student, Teacher, Course, Classroom, Period, Enrollment, Schedule | Academic Service | API + events |
| Class_Session, Attendance, Justification | Attendance Service | API + events |
| Facial_Embedding, Fingerprint_Embedding, Facial_Update_Case | Biometric Service | API + events |
| IoT_Device, Device_Assignment | IoT Service | API + events |
| Audit_Event, Error_Log | Audit Service | API + events |
| Academic_Configuration, Security_Configuration, Password_Policy | Configuration Service | API + events |

No other bounded context should own or directly modify the canonical table rows of another context.

---

## How to add a service

1. Confirm the service corresponds to a bounded context or a distinct domain responsibility.
2. Assign the next available number in the catalog.
3. Create the folder under `09-microservices/services/NN-name/` using the template.
4. Define the local schema and allow only local FK relationships.
5. Publish the public ID references needed by other contexts.
6. Add the service to this catalog and the API contract list.
7. Document the integration events consumed and emitted.

---

## Correlations

- Architecture and domain boundaries → `05-architecture/`
- OpenAPI contracts → `07-api/contracts/openapi/`
- Domain events → `02-domain/domain-events.md`
- Service templates → `09-microservices/_template/service/`
- Per-service implementation documentation → `09-microservices/services/NN-name/`
