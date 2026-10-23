# ADR-005 — Technology Stack Selection

- **ID:** ADR-005
- **Date:** 2026-09-12
- **Status:** Proposed
- **Implementation note (2026-10-05, verified against `FULL/`):** the per-service stack matches the implementation — Java 21 + **Spring Boot 4.1** (01/02/04/05), TypeScript + Fastify + Zod (03/07/**09**), Python 3.12 + FastAPI (06), Go 1.22 + Gin (08), Kong OSS 3.6, PostgreSQL 17, MongoDB 7, Redis 7, Kafka (KRaft). Two deltas: a ninth service **`ms-quality` :8089** (TS, ISO 25010 tooling, not a bounded context) exists, and the modules are separate containers rather than one process (see ADR-002 note). Where the tables below still read `Spring Boot 3` or port `8090` for notifications, read **Spring Boot 4.1** and **8088** (`ms-notification`); the deployed port block is `8081`-`8089`.
- **Superseded in part (2026-10-18):** `08-ms-notification` was **rewritten in TypeScript +
  Fastify + Zod + Drizzle + pino**, so the platform runs **3 runtimes**, not 4 — Go is gone from
  the codebase. The tables and scoring matrices below keep their original Go rows as the historical
  record of the 2026-09-12 decision; the current state is ADR-021. This also closed the tech debt
  `AT-003` ("4th runtime for 2 tables").
- **Authors:** FaceAttend EDU Team

---

## Context

The system consists of 8 bounded-context modules running inside a single modular monolith (per ADR-002), plus an API Gateway. Each module has distinct workload characteristics — from trivial CRUD (configuration-service) to CPU-intensive biometric matching (biometric-service) to ACID transactional attendance recording (attendance-service).

The team must select a concrete technology stack that:
1. Matches the workload profile of each module.
2. Is consistent with the hexagonal architecture standard (`05-architecture/hexagonal-architecture.md`).
3. Minimizes operational complexity for a 3-person team with no dedicated DevOps.
4. Aligns with the documentation already written (examples use TypeScript pseudo-code, `.env.example` has `NODE_ENV`).
5. Supports the infrastructure already decided: PostgreSQL, MongoDB (ADR-003), Redis, Kafka (ADR-006), Liquibase (ADR-010), Jenkins (ADR-011).

**Known constraints:**
- Team of 3 SENA apprentices; learning curve is a hard constraint.
- The documentation (`_stacks/node-typescript.md`, `_stacks/python-fastapi.md`) already provides guides for Node.js/TypeScript and Python/FastAPI.
- Biometric service requires OpenCV + vector similarity search — the Python ecosystem is dominant for computer vision.
- The modular monolith means all modules compile and deploy as a single process.

---

## Decision

We decided on a **polyglot modular monolith** with the following technology stack per service:

| # | Service | Technology | Rationale |
|---|---------|-----------|-----------|
| 01 | `identity-service` (+ auth) | **Java 21 + Spring Boot 3** | Best ORM (Hibernate/JPA) for the 4 entities. Authentication merged into this service (no separate auth-service). Spring Security handles JWT + session management. |
| 02 | `authorization-service` | **Java 21 + Spring Boot 3** | Spring Security is the gold standard for RBAC. Permission checks embedded in JWT claims reduce cross-service calls. |
| 03 | `academic-service` | **TypeScript + Fastify + Drizzle ORM** | Drizzle generates efficient SQL for 8-table JOINs (TypeORM generates bad SQL). Fastify's speed handles complex reporting queries. |
| 04 | `scheduling-service` | **Java 21 + Spring Boot 3** | Unique constraints for anti-double-booking are critical. Hibernate's `@UniqueConstraint` + ACID transactions ensure room/instructor conflict prevention. |
| 05 | `attendance-service` | **Java 21 + Spring Boot 3** | Most critical business path. `@Transactional` ensures ACID for Attendance writes. Spring microservice transport ready for future extraction. |
| 06 | `biometric-service` | **Python 3.12 + FastAPI** | OpenCV canonical binding. face_recognition is Python-first. pymongo vector search. **Non-negotiable — only correct choice for CV workloads.** |
| 07 | `notification-service` | **Go 1.22 + Gin** | Lightweight service for alert lifecycle (2 tables). Go's simplicity and small binary size are ideal for a focused, low-complexity microservice. |
| 08 | `configuration-service` | **TypeScript + Fastify** | Trivial CRUD (3 tables). Fastest to implement. 20 minutes of coding. Minimal dependencies. |
| 99 | `api-gateway` | **Kong OSS** | Battle-tested. Built-in plugins for JWT, rate limiting, CORS. DB-less mode. Already in architecture. |

---

## Scoring methodology

Each option is scored on 5 dimensions (1 = worst, 5 = best):

| Dimension | What it measures |
|-----------|-----------------|
| **Performance** | Throughput, latency, memory footprint for the specific workload |
| **Ecosystem** | Maturity of libraries, community, tutorials for THAT specific use case |
| **Learning Curve** | Time for a team with basic JS/Python/Java knowledge to become productive |
| **Library Fit** | Quality of specific libraries needed (ORM, MQTT client, CV bindings, etc.) |
| **Ops Simplicity** | Deployment, monitoring, debugging, Docker friendliness |

**Total = sum of 5 scores (max 25)**

---

## 1. `identity-service` (+ auth merged) — CRUD, User Management, Authentication

**Port 8081 | PostgreSQL | Core domain**

Workload: Standard CRUD with bulk CSV imports. 4 entities (Person, App_User, User_Session, Password_Policy). Authentication (JWT issuance, refresh token rotation, Redis session management) is merged into this service. **Every authenticated request touches this service.**

| # | Option | Performance | Ecosystem | Learning | Lib Fit | Ops | **Total** | Notes |
|---|--------|:-----------:|:---------:|:--------:|:-------:|:---:|:---------:|-------|
| 1 | **Java + Spring Boot** | 4 | 5 | 2 | 5 | 3 | **19** | Hibernate/JPA best ORM. Spring Security handles auth. JVM overhead but team can learn. |
| 2 | TS + Fastify | 4 | 4 | 5 | 4 | 5 | **22** | Fast, simple. But auth merge needs Redis + JWT libraries that Spring Security already provides. |
| 3 | TS + NestJS | 3 | 5 | 4 | 5 | 4 | **21** | Best DI/structure. Overhead for CRUD + auth is unnecessary. |
| 4 | Python + FastAPI | 3 | 4 | 4 | 4 | 4 | **19** | Pydantic excellent for CSV parsing. Different runtime. |
| 5 | Go + Gin | 5 | 3 | 2 | 3 | 4 | **17** | Fastest. Overkill for CRUD + auth. Team must learn Go. |

**Critical analysis:** Spring Boot wins because the auth-merge decision means this service handles JWT issuance, refresh tokens, and Redis session management. Spring Security provides all of this out-of-the-box with `@EnableWebSecurity`, `JwtEncoder`, and `RedisOperationsSessionRepository`. Building the same in Fastify requires assembling `ioredis` + `jose` + custom middleware — more moving parts, more bugs. The 19/25 score is justified by the combined CRUD + auth workload.

---

## 2. `authorization-service` — RBAC, Permissions

**Port 8083 | PostgreSQL | Core domain**

Workload: Permission checks on **every request**. 4 entities (Role, Permission, Role_Permission, User_Role). Read-heavy (permission lookup) with rare writes (role assignment).

| # | Option | Performance | Ecosystem | Learning | Lib Fit | Ops | **Total** | Notes |
|---|--------|:-----------:|:---------:|:--------:|:-------:|:---:|:---------:|-------|
| 1 | **Java + Spring Boot** | 4 | 5 | 2 | 5 | 3 | **19** | Spring Security is the gold standard for RBAC. Overhead is massive but the team already uses Java for identity-service. |
| 2 | TS + Fastify | 4 | 4 | 5 | 4 | 5 | **22** | Permission checks are DB queries. Fastify handles them fast. Cache in Redis. |
| 3 | TS + NestJS | 3 | 5 | 4 | 5 | 4 | **21** | Guards = RBAC pattern. But permission logic is 10 lines, not 100. |
| 4 | Python + FastAPI | 3 | 4 | 4 | 4 | 4 | **19** | `casbin` Python binding available. Different runtime. |
| 5 | Go + Gin | 5 | 3 | 2 | 3 | 4 | **17** | Fastest permission check. Overkill — cache solves the latency problem. |

**Critical analysis:** The real optimization for authorization is **embedding permissions in JWT claims** (ADR-008), not choosing a faster language. If permissions are in the JWT, authorization-service is never called on most requests. When it IS called, it's a simple DB query cached in Redis with 5-minute TTL. Java wins for consistency with identity-service and Spring Security's built-in RBAC patterns.

---

## 3. `academic-service` — Complex CRUD, Academic Entities

**Port 8084 | PostgreSQL | Core domain**

Workload: 8 entities with relationships. Complex JOIN queries for reports (enrollment lists, schedule conflicts). Reporting is read-heavy. Writes are periodic (semester setup).

| # | Option | Performance | Ecosystem | Learning | Lib Fit | Ops | **Total** | Notes |
|---|--------|:-----------:|:---------:|:--------:|:-------:|:---:|:---------:|-------|
| 1 | **TS + Fastify + Drizzle** | 4 | 4 | 5 | 4 | 5 | **22** | Drizzle generates efficient SQL for complex JOINs. TypeORM generates bad SQL. |
| 2 | Java + Spring Boot | 4 | 5 | 2 | 5 | 3 | **19** | Hibernate handles 8-entity models perfectly. JVM overhead + team learning. |
| 3 | Python + FastAPI + SQLAlchemy | 3 | 4 | 4 | 4 | 4 | **19** | SQLAlchemy excellent for complex queries. GIL limits concurrent report generation. |
| 4 | TS + NestJS + TypeORM | 3 | 5 | 4 | 4 | 4 | **20** | NestJS modules map to entities. But TypeORM generates inefficient JOINs. |
| 5 | Go + Gin + sqlx | 5 | 3 | 2 | 3 | 4 | **17** | Fastest raw queries. But ORM story is weak — manual SQL for everything. |

**Critical analysis:** This is the service where ORM choice matters most. TypeORM's QueryBuilder generates verbose, sometimes incorrect SQL for 8-table JOINs. Drizzle ORM uses a SQL-like API that produces efficient queries without the Hibernate-style overhead. TypeScript + Fastify is the right choice here because Drizzle's type-safety catches schema mismatches at compile time.

---

## 4. `scheduling-service` — Anti-Double-Booking, Unique Constraints

**Port 8087 | PostgreSQL | Core domain**

Workload: 3 entities (Environment, Schedule_Block, Class_Session). **Unique constraints prevent double-booking** — this is the critical invariant. Room and instructor time-slot conflicts must be prevented at the database level.

| # | Option | Performance | Ecosystem | Learning | Lib Fit | Ops | **Total** | Notes |
|---|--------|:-----------:|:---------:|:--------:|:-------:|:---:|:---------:|-------|
| 1 | **Java + Spring Boot** | 4 | 5 | 2 | 5 | 3 | **19** | Hibernate `@UniqueConstraint` + ACID transactions. Spring's `DataIntegrityViolationException` handles constraint conflicts cleanly. Consistent with identity/authorization/attendance. |
| 2 | TS + Fastify + Drizzle | 4 | 4 | 5 | 4 | 5 | **22** | Drizzle unique constraints work. But exception handling for constraint violations is less elegant than Spring's. |
| 3 | TS + NestJS + TypeORM | 3 | 5 | 4 | 4 | 4 | **20** | TypeORM unique constraints. But TypeORM generates inefficient SQL. |
| 4 | Python + FastAPI | 3 | 4 | 4 | 4 | 4 | **19** | SQLAlchemy unique constraints. Different runtime. |
| 5 | Go + Gin | 5 | 3 | 2 | 3 | 4 | **17** | Fastest. But unique constraints are DB-level, not app-level. |

**Critical analysis:** Scheduling-service's critical invariant is preventing double-booking through database unique constraints. Spring Boot's Hibernate handles this with `@UniqueConstraint` on `@Entity` classes and catches `DataIntegrityViolationException` when a conflict occurs — a clean, well-documented pattern. The service also needs ACID transactions to atomically create schedule blocks and class sessions. Java wins for consistency with the other Java services (identity, authorization, attendance).

---

## 5. `attendance-service` — Core Business, Critical Path

**Port 8085 | PostgreSQL | Core domain (critical path)**

Workload: ACID transaction for Attendance write. Biometric match is a precondition (read from MongoDB). This is the most critical business operation. Must be **reliable**, not necessarily the fastest.

| # | Option | Performance | Ecosystem | Learning | Lib Fit | Ops | **Total** | Notes |
|---|--------|:-----------:|:---------:|:--------:|:-------:|:---:|:---------:|-------|
| 1 | **Java + Spring Boot** | 4 | 5 | 2 | 5 | 3 | **19** | `@Transactional` is excellent. Spring microservice transport ready for extraction. Consistent with identity/authorization/scheduling. |
| 2 | TS + Fastify | 4 | 4 | 5 | 4 | 5 | **22** | Faster raw performance. But attendance logic is complex — structure helps. |
| 3 | TS + NestJS | 3 | 5 | 4 | 5 | 4 | **21** | NestJS microservice transport ready for extraction. But Java already provides this. |
| 4 | Python + FastAPI | 3 | 4 | 4 | 4 | 4 | **19** | SQLAlchemy transactions work. But Python GIL could bottleneck concurrent registrations. |
| 5 | Go + Gin | 5 | 3 | 2 | 3 | 4 | **17** | Best reliability. But attendance is I/O-bound (DB), not CPU-bound — Go adds no value. |

**Critical analysis:** Attendance-service is the most critical business path and the first candidate for extraction to a separate service (ADR-002). Spring Boot's `@Transactional` provides declarative ACID management — the same pattern used in banking systems. The ACID transaction is database-level (PostgreSQL), so the language doesn't affect atomicity, but Spring's transaction management is more mature than Node.js alternatives. Consistency with other Java services reduces cognitive load.

---

## 6. `biometric-service` — OpenCV, MongoDB, Vector Embeddings

**Port 8086 | MongoDB (vector search) | Specialized domain**

Workload: CPU-intensive face detection + embedding extraction + vector similarity search. OpenCV is the canonical Python library. This is the one service where language choice is non-negotiable.

| # | Option | Performance | Ecosystem | Learning | Lib Fit | Ops | **Total** | Notes |
|---|--------|:-----------:|:---------:|:--------:|:-------:|:---:|:---------:|-------|
| 1 | **Python + FastAPI** | 4 | 5 | 4 | 5 | 4 | **22** | OpenCV canonical binding. face_recognition is Python-first. pymongo vector search. NumPy native. **This is the only correct choice.** |
| 2 | Python + Flask | 3 | 5 | 4 | 5 | 4 | **21** | Same Python ecosystem. Flask is simpler but less structured than FastAPI. |
| 3 | TS + Fastify | 3 | 2 | 5 | 2 | 4 | **16** | opencv4nodejs is fragile. face-api.js is unmaintained. MongoDB vector search works but bindings are weak. |
| 4 | TS + NestJS | 3 | 2 | 4 | 2 | 4 | **15** | Same as above + framework overhead on CPU-intensive work. |
| 5 | Rust + Actix + opencv-rust | 5 | 2 | 1 | 2 | 3 | **13** | Fastest possible. 12+ month ramp-up. OpenCV Rust bindings exist but are poorly documented. |

**Critical analysis:** Python is non-negotiable here. OpenCV's Python binding is 10x more maintained than any alternative. face_recognition, insightface, facenet-pytorch are all Python-first. MongoDB's official Python driver has native vector search support. The only real choice is FastAPI vs Flask — FastAPI wins because async support handles concurrent face-matching requests better, and auto-generated OpenAPI is useful for the attendance-service integration.

---

## 7. `notification-service` — Alert Lifecycle

**Port 8090 | PostgreSQL | Support domain**

Workload: 2 entities (Alert_Type, Alert). Simple CRUD with alert lifecycle (raised → resolved). Publishes alerts to Kafka for downstream consumption.

| # | Option | Performance | Ecosystem | Learning | Lib Fit | Ops | **Total** | Notes |
|---|--------|:-----------:|:---------:|:--------:|:-------:|:---:|:---------:|-------|
| 1 | **Go + Gin** | 5 | 3 | 2 | 3 | 4 | **17** | Tiny binary, minimal memory footprint. Go's simplicity is ideal for a focused, low-complexity service. Consistent with Go ecosystem if team expands. |
| 2 | TS + Fastify | 4 | 4 | 5 | 4 | 5 | **22** | Fastest to implement. 20 minutes of coding. Minimal dependencies. |
| 3 | TS + NestJS | 3 | 5 | 4 | 5 | 4 | **21** | Structure for 2 tables is overkill. |
| 4 | Python + FastAPI | 3 | 4 | 4 | 4 | 4 | **19** | Auto-gen OpenAPI. Differente runtime for trivial service. |
| 5 | Java + Spring Boot | 4 | 5 | 2 | 5 | 3 | **19** | Consistency with other Java services. But 2 tables don't need Spring. |

**Critical analysis:** Notification-service is the simplest service — 2 tables with basic CRUD and alert lifecycle. Go + Gin is chosen for its minimal resource footprint and simplicity. The tiny binary size (5-10 MB) and low memory usage (5-10 MB) make it ideal for a service that will likely be deployed as a sidecar or lightweight container. Go's simplicity also provides a good learning opportunity for the team without the complexity of Spring Boot.

---

## 8. `configuration-service` — Simple CRUD, Config Values

**Port 8089 | PostgreSQL | Support domain**

Workload: 2 tables (Academic_Configuration, Security_Configuration). Basic CRUD. Published config changes to Kafka. **Trivial service.**

| # | Option | Performance | Ecosystem | Learning | Lib Fit | Ops | **Total** | Notes |
|---|--------|:-----------:|:---------:|:--------:|:-------:|:---:|:---------:|-------|
| 1 | **TS + Fastify** | 4 | 4 | 5 | 4 | 5 | **22** | Fastest to implement. 20 minutes of coding. Minimal dependencies. |
| 2 | TS + NestJS | 3 | 5 | 4 | 5 | 4 | **21** | Auto-generated OpenAPI via decorators. Overkill for 3 tables. |
| 3 | Python + FastAPI | 3 | 4 | 4 | 4 | 4 | **19** | Auto-generated OpenAPI. Different runtime for the simplest service. |
| 4 | Go + Gin | 5 | 3 | 2 | 3 | 4 | **17** | Fastest. But we're talking about 3 config tables. |
| 5 | Express (TS) | 3 | 4 | 5 | 3 | 5 | **20** | Simplest possible. No structure needed. But no OpenAPI generation. |

**Critical analysis:** This service has no domain logic. It's 2 Postgres tables with CRUD endpoints. Fastify is the lightest framework that still gives good performance. NestJS's decorators for auto-generated OpenAPI are nice but not worth the overhead for a service that will never change.

---

## 10. `api-gateway` — Routing, Rate Limiting, Auth Enforcement

**Port 8000 | Redis (cache) | Infrastructure**

Workload: Routes external requests to the single monolith application. JWT validation, rate limiting, CORS. Must be fast and reliable.

| # | Option | Performance | Ecosystem | Learning | Lib Fit | Ops | **Total** | Notes |
|---|--------|:-----------:|:---------:|:--------:|:-------:|:---:|:---------:|-------|
| 1 | **Kong OSS** | 5 | 5 | 3 | 5 | 4 | **22** | Battle-tested. Built-in plugins for JWT, rate limiting, CORS. DB-less mode. Already in architecture. |
| 2 | Traefik | 4 | 4 | 3 | 4 | 5 | **20** | Modern, auto-discovery with Docker. Less enterprise plugins than Kong. |
| 3 | NGINX + Lua | 5 | 4 | 2 | 3 | 3 | **17** | Fastest possible. But Lua scripting for custom logic is painful. |
| 4 | Caddy | 4 | 3 | 3 | 3 | 5 | **18** | Auto-HTTPS, simple config. Less plugin ecosystem. |
| 5 | Node.js + Fastify (custom) | 3 | 4 | 5 | 4 | 4 | **20** | Full TypeScript control. But reinventing rate limiting and JWT validation is wasteful. |

**Critical analysis:** The API Gateway is infrastructure, not application code. Kong is already decided in the architecture. Building a custom gateway in Node.js means reimplementing rate limiting, JWT validation, CORS, and SSL termination — all problems Kong solves out of the box. NGINX is faster but requires Lua for custom logic. Traefik is modern but Kong has stronger enterprise adoption.

---

## Summary: Final technology map

```
┌─────────────────────────────────────────────────────────────────┐
│                     API Gateway (Kong OSS)                      │
│                     Port 8000 | Redis                           │
└──────────────────────────┬──────────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────────┐
│                    Modular Monolith (JVM)                        │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐ │
│  │ identity-service │  │ authorization-  │  │ scheduling-     │ │
│  │ (+ auth merged)  │  │ service         │  │ service         │ │
│  │ Java + Spring    │  │ Java + Spring   │  │ Java + Spring   │ │
│  │ Port 8081        │  │ Port 8083       │  │ Port 8087       │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘ │
│  ┌─────────────────┐                                           │
│  │ attendance-     │                                           │
│  │ service         │                                           │
│  │ Java + Spring   │                                           │
│  │ Port 8085       │                                           │
│  └─────────────────┘                                           │
├─────────────────────────────────────────────────────────────────┤
│                    Modular Monolith (Node.js)                    │
│  ┌─────────────────┐  ┌─────────────────┐                       │
│  │ academic-service │  │ configuration-  │                       │
│  │ TS + Fastify     │  │ service         │                       │
│  │ + Drizzle        │  │ TS + Fastify    │                       │
│  │ Port 8084        │  │ Port 8089       │                       │
│  └─────────────────┘  └─────────────────┘                       │
├─────────────────────────────────────────────────────────────────┤
│                    Modular Monolith (Python)                     │
│  ┌─────────────────┐                                           │
│  │ biometric-      │                                           │
│  │ service         │                                           │
│  │ Python + FastAPI│                                           │
│  │ Port 8086       │                                           │
│  └─────────────────┘                                           │
├─────────────────────────────────────────────────────────────────┤
│                    Modular Monolith (Go)                         │
│  ┌─────────────────┐                                           │
│  │ notification-   │                                           │
│  │ service         │                                           │
│  │ Go + Gin        │                                           │
│  │ Port 8090       │                                           │
│  └─────────────────┘                                           │
└─────────────────────────────────────────────────────────────────┘
```

---

## Key findings

1. **Java Spring Boot dominates core business** (4 services: identity, authorization, scheduling, attendance). The learning curve (2/5) is justified by the best ORM (Hibernate), transaction management (`@Transactional`), and Spring Security for RBAC. Consistency across core services reduces cognitive load.

2. **TypeScript + Fastify for reporting and support** (2 services: academic, configuration). Drizzle ORM's SQL-like API produces better queries than TypeORM for complex JOINs. Fastify's speed handles reporting workloads.

3. **Python for CV only** (1 service: biometric). OpenCV is non-negotiable in Python. FastAPI's async handles concurrent face-matching.

4. **Go for lightweight services** (1 service: notification). Go's minimal footprint and simplicity are ideal for a focused, low-complexity service.

5. **Kong for infrastructure** (1 service: api-gateway). Battle-tested, plugin ecosystem, already in architecture.

---

## Consequences

1. **Team must learn Java/Spring Boot** for 4 core services. This is a 3-6 month learning curve but justified by the enterprise patterns needed for the most critical business paths.

2. **4 runtimes in production**: JVM (Java), Node.js (TypeScript), Python, Go. This increases operational complexity — 4 Docker base images, 4 dependency management systems, 4 debugging approaches.

3. **TypeScript + Fastify stack is validated** for reporting and support services. Drizzle ORM is the recommended choice for complex queries.

4. **Python + FastAPI is validated** only for CV workloads. Not recommended for other services.

5. **Go + Gin is validated** for lightweight, focused services. Not recommended for complex business logic.

6. **Modular monolith extraction path**: The 4 Java services can be extracted together (same runtime, same Spring patterns). The 2 TypeScript services can be extracted together. Python and Go services are standalone.

---

## Risks

1. **Java learning curve** (High): Team of 3 SENA apprentices must learn Java, Spring Boot, Hibernate, and Maven/Gradle. Mitigation: Start with identity-service (simplest Java service) and progress to more complex services.

2. **4-runtime operational complexity** (Medium): Managing JVM, Node.js, Python, and Go in production requires different Docker images, monitoring approaches, and debugging tools. Mitigation: Standardize on Docker + Jenkins pipeline per runtime.

3. **Spring Boot overhead for simple services** (Low): Authorization-service and scheduling-service could be simpler in TypeScript. Mitigation: The consistency benefit outweighs the overhead — 4 services sharing the same patterns is better than 4 different approaches.

4. **Go learning curve for notification-service** (Low): Team must learn Go basics. Mitigation: notification-service is the simplest service (2 tables), making it a low-risk Go learning opportunity.

5. **Drizzle ORM maturity** (Low): Drizzle is newer than TypeORM. Mitigation: Drizzle's SQL-like API is well-documented and has strong community support.

---

## References

- Hexagonal Architecture standard → `05-architecture/hexagonal-architecture.md`
- Stack guides → `_stacks/node-typescript.md`, `_stacks/python-fastapi.md`
- Related to: ADR-002 (Modular Monolith), ADR-003 (Biometric persistence in MongoDB)
