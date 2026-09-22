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
                     │ 01 Identity (+auth)  │
                     │ 02 Authorization     │
                     │ 03 Academic          │
                     │ 04 Scheduling        │
                     │ 05 Attendance        │
                     │ 06 Biometric         │
                      │ 07 Notification      │
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
| 01 | `identity-service` (+ auth) | Person, user, authentication, sessions, JWT | 8081 | PostgreSQL | Relational | 🟢 Active |
| 02 | `authorization-service` | Roles, permissions, and role assignment policies | 8083 | PostgreSQL | Relational | 🟢 Active |
| 03 | `academic-service` | Students, teachers, courses, periods, enrollments | 8084 | PostgreSQL | Relational | 🟢 Active |
| 04 | `scheduling-service` | Environments, schedule blocks, class sessions | 8087 | PostgreSQL | Relational | 🟢 Active |
| 05 | `attendance-service` | Class sessions, attendance, and justification processing | 8085 | PostgreSQL | Relational | 🟢 Active |
| 06 | `biometric-service` | Facial and fingerprint embeddings and update review cases | 8086 | MongoDB | NoSQL / Document | 🟢 Active |
| 07 | `notification-service` | Alert types and alert lifecycle management | 8090 | PostgreSQL | Relational | 🟢 Active |
| 08 | `configuration-service` | Academic and security configuration values and password policy | 8089 | PostgreSQL | Relational | 🟢 Active |
| 99 | `api-gateway` | Edge routing and security enforcement for external clients | 8080 | Redis | Cache | 🟢 Active |

**Status legend:** 🟢 Active | 🟡 In development | 🔴 Planned | ⏸ Deprecated

---

## Detail per service

### 01 — `identity-service` (+ auth merged)

| Field | Value |
|-------|-------|
| **Folder** | `09-microservices/services/01-identity-service/` |
| **Responsibility** | Owns person profiles, user credentials, authentication sessions, JWT issuance, and refresh token management |
| **Type** | Core domain service |
| **Port** | 8081 |
| **DB** | PostgreSQL |
| **Bounded context** | Identity (includes Authentication) |
| **Technology** | Java 21 + Spring Boot 3 |
| **Externally exposed** | Yes, via API Gateway |

**Owned entities:**
- `School`
- `Person`
- `App_User`
- `User_Session`
- JWT access and refresh tokens
- Redis session management

**Cross-context rule:**
- `Person.school_id` and `User.person_id` remain internal to Identity.
- Authorization and other services use public IDs only.
- Authentication is merged from the former `auth-service` — no separate auth microservice.

---

### 02 — `authorization-service`

| Field | Value |
|-------|-------|
| **Folder** | `09-microservices/services/02-authorization-service/` |
| **Responsibility** | Defines roles, permissions, and assignments that authorize access across the platform |
| **Type** | Core domain service |
| **Port** | 8083 |
| **DB** | PostgreSQL |
| **Bounded context** | Authorization |
| **Technology** | Java 21 + Spring Boot 3 |
| **Externally exposed** | Yes, via API Gateway |

**Owned entities:**
- `Role`
- `Permission`
- `Role_Permission`
- `User_Role`

**Cross-context rule:**
- `User_Role.user_id` references the public `Identity.User.user_id` without a direct FK.

---

### 03 — `academic-service`

| Field | Value |
|-------|-------|
| **Folder** | `09-microservices/services/03-academic-service/` |
| **Responsibility** | Manages students, teachers, courses, periods, enrollments, and the academic structure |
| **Type** | Core domain service |
| **Port** | 8084 |
| **DB** | PostgreSQL |
| **Bounded context** | Academic |
| **Technology** | TypeScript + Fastify + Drizzle ORM |
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

### 04 — `scheduling-service`

| Field | Value |
|-------|-------|
| **Folder** | `09-microservices/services/04-scheduling-service/` |
| **Responsibility** | Manages physical environments, schedule blocks, and class sessions with anti-double-booking constraints |
| **Type** | Core domain service |
| **Port** | 8087 |
| **DB** | PostgreSQL |
| **Bounded context** | Scheduling |
| **Technology** | Java 21 + Spring Boot 3 |
| **Externally exposed** | Yes, via API Gateway |

**Owned entities:**
- `Environment`
- `Schedule_Block`
- `Class_Session`

**Cross-context rule:**
- `Environment.school_id` is an external reference to Academic.school.
- `Schedule_Block.cohort_id`, `course_id`, `instructor_actor_id` are external references to Academic.
- `Class_Session.opened_by` and `closed_by` are external references to Academic.

---

### 05 — `attendance-service`

| Field | Value |
|-------|-------|
| **Folder** | `09-microservices/services/05-attendance-service/` |
| **Responsibility** | Records actual class sessions, individual attendance data, and justification handling |
| **Type** | Core domain service (critical path) |
| **Port** | 8085 |
| **DB** | PostgreSQL |
| **Bounded context** | Attendance |
| **Technology** | Java 21 + Spring Boot 3 |
| **Externally exposed** | Yes, via API Gateway |

**Owned entities:**
- `Class_Session`
- `Attendance_Record`
- `Justification_Type`
- `Justification`
- `Supporting_Document`

**Cross-context rule:**
- `student_id`, `schedule_id`, and `opened_by` are public references without FKs to other contexts.

---

### 06 — `biometric-service`

| Field | Value |
|-------|-------|
| **Folder** | `09-microservices/services/06-biometric-service/` |
| **Responsibility** | Stores biometric templates and review cases for recognition workflows |
| **Type** | Specialized domain service |
| **Port** | 8086 |
| **DB** | MongoDB (document store with vector search indexing) |
| **Bounded context** | Biometric |
| **Technology** | Python 3.12 + FastAPI |
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

### 07 — `notification-service`

| Field | Value |
|-------|-------|
| **Folder** | `09-microservices/services/07-notification-service/` |
| **Responsibility** | Manages alert types and alert lifecycle for system-wide notifications |
| **Type** | Support domain service |
| **Port** | 8090 |
| **DB** | PostgreSQL |
| **Bounded context** | Notification |
| **Technology** | Go 1.22 + Gin |
| **Externally exposed** | Yes, via API Gateway |

**Owned entities:**
- `Alert_Type`
- `Alert`

**Cross-context rule:**
- `Alert.academic_actor_id` is an external reference to Academic — no cross-context FK.

---

### 08 — `configuration-service`

| Field | Value |
|-------|-------|
| **Folder** | `09-microservices/services/07-configuration-service/` |
| **Responsibility** | Holds academic, security, and password-related configuration values used by the platform |
| **Type** | Support domain service |
| **Port** | 8089 |
| **DB** | PostgreSQL |
| **Bounded context** | Configuration |
| **Technology** | TypeScript + Fastify |
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
| **Folder** | `09-microservices/services/99-api-gateway/` |
| **Responsibility** | Routes requests, enforces authentication and policy boundaries, and exposes the platform to clients |
| **Type** | Infrastructure service |
| **Port** | 8080 |
| **DB** | Redis |
| **Bounded context** | None; cross-cutting infrastructure |
| **Technology** | Kong OSS |
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
| API Gateway | Scheduling Service | HTTP/REST | Synchronous |
| API Gateway | Attendance Service | HTTP/REST | Synchronous |
| API Gateway | Biometric Service | HTTP/REST | Synchronous |
| API Gateway | Notification Service | HTTP/REST | Synchronous |
| API Gateway | Configuration Service | HTTP/REST | Synchronous |
| Identity Service | Authorization Service | Event | Asynchronous |
| Identity Service | Academic Service | Event | Asynchronous |
| Identity Service | Configuration Service | Event | Asynchronous |
| Academic Service | Scheduling Service | Event | Asynchronous |
| Academic Service | Attendance Service | Event | Asynchronous |
| Scheduling Service | Attendance Service | Event | Asynchronous |
| Biometric Service | Attendance Service | Event | Asynchronous |
| Configuration Service | Academic Service | Event | Asynchronous |
| Configuration Service | Identity Service | Event | Asynchronous |
| Attendance Service | Notification Service | Event | Asynchronous |

---

## Data ownership matrix

| Domain entity | Source of truth | Access pattern |
|---------------|-----------------|----------------|
| School, Person, User, User_Session | Identity Service | API + events |
| Role, Permission, User_Role | Authorization Service | API + events |
| Student, Teacher, Course, Classroom, Period, Enrollment | Academic Service | API + events |
| Environment, Schedule_Block, Class_Session | Scheduling Service | API + events |
| Attendance_Record, Justification_Type, Justification, Supporting_Document | Attendance Service | API + events |
| Facial_Embedding, Fingerprint_Embedding, Facial_Update_Case | Biometric Service | API + events |
| Alert_Type, Alert | Notification Service | API + events |
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
