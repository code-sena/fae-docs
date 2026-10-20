# Microservices Catalog

> This catalog reflects the current bounded-context model: each service owns a business domain and the data within it, while cross-context references remain public IDs and event-driven integration.

---

## Service map

```
                     ┌──────────────────────┐
  External clients    │ API Gateway / Edge   │
  (Web, Mobile, API) ─▶│ Kong :8000           │
                     │ Session check + CORS │
                     │ (no jwt plugin)      │
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
                     │ 07 Configuration     │
                     │ 08 Notification      │
                     │ 09 Quality (tooling) │
                     └──────────┬───────────┘
                                │
                     ┌──────────▼───────────┐
                     │ Event transport      │
                     │ Kafka (KRaft :9092)  │
                     └──────────────────────┘
```

> The API gateway is infrastructure. The business model itself is formed by the eight bounded contexts above; `ms-quality` (09) is tooling owned by `11-quality`, not a bounded context.

---

## Service registry

| # | Service | Main responsibility | Port | DB | DB type | Status |
|---|---------|-------------------|------|-----|---------|--------|
| 01 | `ms-identity` (+ auth) | Person, user, authentication (opaque `sessionId`), sessions | 8081 | PostgreSQL | Relational | 🟢 Active |
| 02 | `ms-authorization` | Roles, permissions, and role assignment policies | 8082 | PostgreSQL | Relational | 🟢 Active |
| 03 | `ms-academic` | Schools, actors, courses, periods, enrollments | 8083 | PostgreSQL | Relational | 🟢 Active |
| 04 | `ms-scheduling` | Environments, schedule blocks, class sessions | 8084 | PostgreSQL | Relational | 🟢 Active |
| 05 | `ms-attendance` | Class sessions attendance and justification processing | 8085 | PostgreSQL | Relational | 🟢 Active |
| 06 | `ms-biometric` | Facial and fingerprint embeddings and update review cases | 8086 | MongoDB (+ `biometric` schema) | NoSQL / Document | 🟢 Active |
| 07 | `ms-configuration` | Academic and security configuration values, biometric update cases | 8087 | PostgreSQL | Relational | 🟢 Active |
| 08 | `ms-notification` | Alert types and alert lifecycle management | 8088 | PostgreSQL | Relational | 🟢 Active |
| 09 | `ms-quality` (tooling) | ISO 25010/29110/ISTQB quality metrics | 8089 | PostgreSQL (`quality`) | Relational | 🟢 Active |
| 99 | `kong-gateway` | Edge routing, session check, CORS and rate limiting (**no jwt**) | 8000 / 8001 | Redis | Rate-limit store | 🟢 Active |

**Status legend:** 🟢 Active | 🟡 In development | 🔴 Planned | ⏸ Deprecated

> Ports frozen from `FULL/COMPOSE.md` + `FULL/back-end/99-api-gateway/kong/kong.yml`.

---

## Detail per service

### 01 — `ms-identity` (+ auth merged)

| Field | Value |
|-------|-------|
| **Folder (docs)** | `09-microservices/services/01-identity-service/` |
| **Folder (repo)** | `FULL/back-end/01-ms-identity/` |
| **Responsibility** | Owns person profiles, user credentials, authentication sessions, session-id issuance, and password policy |
| **Type** | Core domain service |
| **Port** | 8081 |
| **DB** | PostgreSQL, schema `identity` |
| **Bounded context** | Identity (includes Authentication) |
| **Technology** | Java 21 + Spring Boot 4.1 |
| **Externally exposed** | Yes, via API Gateway |

**Owned entities:**
- `Person`
- `App_User`
- `City` (**removed** — dropped by Liquibase; cities come from the external `countriesnow.space` API)
- `User_Session`
- `Password_Policy` (singleton global)

**Authentication reality:**
- Login returns an **opaque `sessionId`** (row in `identity.user_session`); it is sent back as
  `Authorization: Bearer <sessionId>` and checked **at the edge** by Kong's global `pre-function`
  (introspection against identity) and then by every service's own guard, which also applies RBAC
  — `AuthTokenFilter` in Java (01/02/04/05), `registerAuthGuard` in TS (03/07/08/09),
  `require_permission` in Python (06). There are **no JWT
  access/refresh tokens and no Redis session store** — Kong carries no `jwt` plugin. See
  `07-api/authentication.md`.

**Cross-context rule:**
- `Person` has no `school_id` — multi-tenancy via `Academic.academic_actor`.
- `App_User.person_id` is internal; other services use public `user_id`/`person_id` UUIDs.
- Authentication is merged from the former `auth-service` — no separate auth microservice.

---

### 02 — `ms-authorization`

| Field | Value |
|-------|-------|
| **Folder (docs)** | `09-microservices/services/02-authorization-service/` |
| **Folder (repo)** | `FULL/back-end/02-ms-authorization/` |
| **Responsibility** | Defines roles, permissions, and assignments that authorize access across the platform |
| **Type** | Core domain service |
| **Port** | 8082 |
| **DB** | PostgreSQL, schema `authorization` |
| **Bounded context** | Authorization |
| **Technology** | Java 21 + Spring Boot 4.1 |
| **Externally exposed** | Yes, via API Gateway |

**Owned entities:**
- `Role`
- `Permission`
- `Role_Permission`
- `User_Role`

**Cross-context rule:**
- `User_Role.user_id` references the public `Identity.User.user_id` without a direct FK.

---

### 03 — `ms-academic`

| Field | Value |
|-------|-------|
| **Folder (docs)** | `09-microservices/services/03-academic-service/` |
| **Folder (repo)** | `FULL/back-end/03-ms-academic/` |
| **Responsibility** | Manages schools, programs, periods, cohorts, courses, actors and enrollments |
| **Type** | Core domain service |
| **Port** | 8083 |
| **DB** | PostgreSQL, schema `academic` |
| **Bounded context** | Academic |
| **Technology** | TypeScript + Fastify + Zod (Drizzle in the model layer) |
| **Externally exposed** | Yes, via API Gateway |

**Owned entities:**
- `School`
- `Program`
- `Academic_Period`
- `Cohort`
- `Course`
- `Academic_Actor_Type`
- `Academic_Actor`
- `Enrollment`

**Cross-context rule:**
- `Academic_Actor.person_id` is a public ID (→ Identity.person), not a local FK.
- `School.city_id` is a public ID (→ external `countriesnow.space` catalog; the local `identity.city` table was dropped).
- `Academic_Actor` unifies Student/Instructor via `actor_type_id` — no separate tables.

---

### 04 — `ms-scheduling`

| Field | Value |
|-------|-------|
| **Folder (docs)** | `09-microservices/services/04-scheduling-service/` |
| **Folder (repo)** | `FULL/back-end/04-ms-scheduling/` |
| **Responsibility** | Manages physical environments, schedule blocks, and class sessions with anti-double-booking constraints |
| **Type** | Core domain service |
| **Port** | 8084 |
| **DB** | PostgreSQL, schema `scheduling` |
| **Bounded context** | Scheduling |
| **Technology** | Java 21 + Spring Boot 4.1 |
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

### 05 — `ms-attendance`

| Field | Value |
|-------|-------|
| **Folder (docs)** | `09-microservices/services/05-attendance-service/` |
| **Folder (repo)** | `FULL/back-end/05-ms-attendance/` |
| **Responsibility** | Records actual class sessions, individual attendance data, and justification handling |
| **Type** | Core domain service (critical path) |
| **Port** | 8085 |
| **DB** | PostgreSQL, schema `attendance` |
| **Bounded context** | Attendance |
| **Technology** | Java 21 + Spring Boot 4.1 |
| **Externally exposed** | Yes, via API Gateway |

**Owned entities:**
- `Attendance_Record`
- `Justification_Type`
- `Justification`
- `Supporting_Document`

**Cross-context rule:**
- `Attendance_Record.class_session_id` → Scheduling.class_session (BIGSERIAL external)
- `Attendance_Record.academic_actor_id` → Academic.academic_actor (BIGSERIAL external)
- `Justification.reviewed_by` → Identity.app_user (UUID external)
- No physical FK across services (Database per Service).

---

### 06 — `ms-biometric`

| Field | Value |
|-------|-------|
| **Folder (docs)** | `09-microservices/services/06-biometric-service/` |
| **Folder (repo)** | `FULL/back-end/06-ms-biometric/` |
| **Responsibility** | Stores biometric templates and review cases for recognition workflows |
| **Type** | Specialized domain service |
| **Port** | 8086 |
| **DB** | MongoDB (document store with vector search indexing) |
| **Bounded context** | Biometric |
| **Technology** | Python 3.12 + FastAPI + Motor |
| **Externally exposed** | Yes, via API Gateway |

**Owned entities:**
- `Facial_Embedding` (MongoDB `facial_embeddings`)
- `Fingerprint_Embedding` (MongoDB `fingerprint_embeddings`)

**Cross-context rule:**
- `person_id` is a public identity reference (→ Identity.person), not a foreign key.
- `biometric_update_case` is **not** owned here — lives in Configuration service (SQL) and references these collections by logical `_id`. In practice `ms-biometric` also runs its own independent `/api/v1/biometric/update-request*` workflow against a local MongoDB `biometric_update_cases` collection, unsynced with the canonical one (`R-004`, see `08-uml/diagrams/source/seq-biometric-update-duplication.mmd`).

**Database note:** Per ADR-003, Biometric uses MongoDB instead of PostgreSQL. This is a
documented exception to the single-database strategy in ADR-002, justified by the specialized
vector search requirements of biometric embedding data.

---

### 07 — `ms-configuration`

| Field | Value |
|-------|-------|
| **Folder (docs)** | `09-microservices/services/07-configuration-service/` |
| **Folder (repo)** | `FULL/back-end/07-ms-configuration/` |
| **Responsibility** | Holds academic and security configuration values, and hosts the biometric update workflow |
| **Type** | Support domain service |
| **Port** | 8087 |
| **DB** | PostgreSQL, schema `configuration` |
| **Bounded context** | Configuration |
| **Technology** | TypeScript + Fastify + Zod |
| **Externally exposed** | Yes, via API Gateway |

**Owned entities:**
- `Academic_Configuration` (per-school)
- `Security_Configuration` (global)
- `Biometric_Update_Case` (unified FACIAL/FINGERPRINT workflow, Pending→In_Review→Approved/Rejected)

**Cross-context rule:**
- `Academic_Configuration.school_id` → Academic.school (external)
- `Biometric_Update_Case.person_id` / `requested_by` / `reviewed_by` → Identity (external)
- `Biometric_Update_Case.current_embedding_ref` → Biometric Mongo `_id` (cross-paradigm, VARCHAR logical)
- `Password_Policy` is owned by Identity service (not here) — singleton global.

---

### 08 — `ms-notification`

| Field | Value |
|-------|-------|
| **Folder (docs)** | `09-microservices/services/08-notification-service/` |
| **Folder (repo)** | `FULL/back-end/08-ms-notification/` |
| **Responsibility** | Manages alert types and alert lifecycle for system-wide notifications |
| **Type** | Support domain service |
| **Port** | 8088 |
| **DB** | PostgreSQL, schema `notification` |
| **Bounded context** | Notification |
| **Technology** | TypeScript + Fastify + Zod + Drizzle + pino (rewritten from Go/Gin 2026-10-18, ADR-021) |
| **Also consumes** | Kafka topic `attendance.justification.events` (group `notification-service`) — the platform's only consumer |
| **Externally exposed** | Yes, via API Gateway |

**Owned entities:**
- `Alert_Type`
- `Alert`

**Cross-context rule:**
- `Alert.academic_actor_id` is an external reference to Academic — no cross-context FK.

---

### 09 — `ms-quality` (tooling, not a bounded context)

| Field | Value |
|-------|-------|
| **Folder (docs)** | `09-microservices/services/09-quality-service/` |
| **Folder (repo)** | `FULL/back-end/09-ms-quality/` |
| **Responsibility** | ISO 25010 / ISO 29110 / ISTQB quality metrics and assessments |
| **Type** | Tooling (owned by `11-quality`) |
| **Port** | 8089 |
| **DB** | PostgreSQL, schema `quality` |
| **Bounded context** | None — excluded from the 8 product contexts |
| **Technology** | TypeScript + Fastify + Zod |
| **Externally exposed** | Yes, via API Gateway (`/api/v1/quality`, rate limit 6000/min) |

**Owned entities:**
- `Quality_Project`, `Quality_Evaluation`, `Quality_Evaluation_Item`
- `Process_Assessment`, `Process_Assessment_Rating`
- `ISTQB_Assessment`, `ISTQB_Assessment_Item`

---

### 99 — `kong-gateway`

| Field | Value |
|-------|-------|
| **Folder (repo)** | `FULL/back-end/99-api-gateway/` (`kong/kong.yml`) |
| **Responsibility** | Routes requests to the 9 services, applies CORS and rate limits, exposes the platform to clients |
| **Type** | Infrastructure service |
| **Port** | 8000 (proxy) / 8001 (admin) |
| **DB** | Redis (rate-limiting store) |
| **Bounded context** | None; cross-cutting infrastructure |
| **Technology** | Kong OSS 3.6, DB-less declarative config |
| **Externally exposed** | Yes |

**Notes:**
- The gateway is not a business owner in the bounded-context model.
- **Authentication is enforced at the edge here**: a global `pre-function` (Lua) requires a valid
  active session on every non-public route — no `jwt` plugin and no consumers, the token is an
  opaque `sessionId` introspected against identity. Every service checks it again and adds RBAC
  on its own — not just the Java ones: `AuthTokenFilter` (01/02/04/05), `registerAuthGuard`
  (03/07/08/09), `require_permission` (06), three duplicated implementations
  (`R-003`, corrected 2026-10-11); see `07-api/authentication.md`.
- The only health route routed at the edge is `GET /health/quality` → `ms-academic:8083`.
- Routes match by path prefix (`/api/v1/persons`, `/api/v1/users`, `/api/v1/auth`, …); there is
  no gateway-level `/health`.

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
| API Gateway | Quality Service | HTTP/REST | Synchronous |
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
| Person, App_User, User_Session, Password_Policy | Identity Service | API + events |
| Role, Permission, User_Role | Authorization Service | API + events |
| School, Program, Academic_Period, Cohort, Course, Academic_Actor, Enrollment | Academic Service | API + events |
| Environment, Schedule_Block, Class_Session | Scheduling Service | API + events |
| Attendance_Record, Justification_Type, Justification, Supporting_Document, Attendance_Report | Attendance Service | API + events |
| Facial_Embedding, Fingerprint_Embedding (MongoDB) | Biometric Service | API |
| Academic_Configuration, Security_Configuration, Biometric_Update_Case | Configuration Service | API + events |
| Alert_Type, Alert | Notification Service | API + events |
| Quality/ISTQB assessments (`quality` schema) | Quality Service | API |

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
