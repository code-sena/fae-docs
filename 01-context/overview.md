# General Context — FaceAttend EDU

## Institutional Context

FaceAttend EDU is developed within the **Software Analysis and Development program — SENA**, as a technological solution to support attendance management in educational environments (initial pilot: one institution, 30+ learners per instructor). See `01-context/scope.md` for boundaries and `00-governance/agile-conventions.md` for team cadence.

## What is FaceAttend EDU?

**FaceAttend EDU** is a **web + mobile (responsive)** attendance-management platform that **automates entry/exit recording via facial recognition** with **manual/import fallback**, centralizes attendance histories, and provides **justification workflows, role-based reports and alerts**. It runs as **9 microservices behind Kong (schema-per-context, 8 bounded contexts) — Kafka events live**, per `05-architecture/overview.md` and `ADR-002/006`.

## Problem it solves

In many academic processes, attendance depends on a single instructor taking manual lists. This causes **inconsistencies, administrative overload, impersonation risk, and delayed follow-up** on absences/late arrivals, especially with **30+ learners per cohort** (SRS §1.c justification).

## Main users

- **Student / Learner:** records attendance contactlessly, consults personal history, submits justifications.
- **Instructor / Teacher:** opens/closes Class_Sessions, views cohort attendance, receives late/absence alerts, reviews justifications.
- **Administrator:** manages users/roles/permissions, schools/programs/cohorts/environments, and system configuration (per-school and global).
- **System (future IoT):** external supporting actor for classroom hardware — out of MVP core (see `scope.md`).

## Technology stack (v1.2, 8 bounded contexts + quality tooling)

- **Backend:** 9 services in 3 runtimes — Java 21 + Spring Boot 4.1 (`ms-identity`, `ms-authorization`, `ms-scheduling`, `ms-attendance`), TypeScript + Fastify + Zod (`ms-academic`, `ms-configuration`, `ms-notification`, `ms-quality`), Python 3.12 + FastAPI (`ms-biometric`)
- **Databases:** Single logical DB `faceattend_db` on PostgreSQL 17 (**8 schemas**) + MongoDB 7 (facial/fingerprint vectors) + Redis 7 (Kong rate-limiting store)
- **Edge:** Kong OSS 3.6 :8000 (proxy) / :8001 (admin) — global `pre-function` session check (introspection against identity) + cors + rate-limiting, **no jwt plugin**; RBAC stays in the Java services (`07-api/authentication.md`)
- **Events:** Kafka (KRaft) `:9092` — real broker, producers via `KafkaTemplate` / `kafkajs` (ADR-006)
- **Infrastructure:** Docker Compose (root `docker-compose.yml` + `COMPOSE.md`), Liquibase migrations (8 jobs, ADR-010), Jenkins CI/CD (ADR-011)

## Main architecture — Bounded Contexts & Services (Schema per Context, Single DB)

> **Ubiquitous Language:** service names below are the single source of truth — identical in `02-domain/domain-map.md`, `05-architecture/overview.md`, `09-microservices/service-catalog.md` and `06-data/models.md` (Informe 2026-09-22 §3.1-§3.2). Ports frozen from `FULL/COMPOSE.md` + `kong.yml`.

| # | Service | Bounded Context | Main responsibility | Port | DB | Technology |
|---|---|---|---|---|---|---|
| 01 | `ms-identity` | Identity | Person, App_User, User_Session, Password_Policy, auth (opaque `sessionId`) (City legacy via external API) | 8081 | PostgreSQL | Java 21 + Spring Boot 4.1 |
| 02 | `ms-authorization` | Authorization | Role, Permission, Role_Permission, User_Role (global roles) | 8082 | PostgreSQL | Java 21 + Spring Boot 4.1 |
| 03 | `ms-academic` | Academic | School, Program, Academic_Period, Cohort, Course, Academic_Actor, Enrollment | 8083 | PostgreSQL | TypeScript + Fastify |
| 04 | `ms-scheduling` | Scheduling | Environment, Schedule_Block, Class_Session (anti-double-booking) | 8084 | PostgreSQL | Java 21 + Spring Boot 4.1 |
| 05 | `ms-attendance` | Attendance | Attendance_Record, Justification_Type, Justification, Supporting_Document, Attendance_Report (derived) | 8085 | PostgreSQL | Java 21 + Spring Boot 4.1 |
| 06 | `ms-biometric` | Biometric | Facial_Embedding, Fingerprint_Embedding (MongoDB vectors) + Biometric_Update_Case | 8086 | **MongoDB + SQL case** | Python 3.12 + FastAPI |
| 07 | `ms-configuration` | Configuration | Academic_Configuration, Security_Configuration (+ hosts biometric update cases) | 8087 | PostgreSQL | TypeScript + Fastify |
| 08 | `ms-notification` | Notification | Alert_Type, Alert; consumes `attendance.justification.events` (Kafka) to raise review alerts | 8088 | PostgreSQL | TypeScript + Fastify |
| 09 | `ms-quality` | — (tooling, not a bounded context) | ISO 25010/29110/ISTQB quality metrics | 8089 | PostgreSQL | TypeScript + Fastify |
| 99 | `kong-gateway` | — (infrastructure) | Routing, edge session check, CORS, rate limiting (no jwt) | 8000 / 8001 | Redis (rate-limit store) | Kong OSS 3.6 |
| — | `frontend-web` | — | Expo Web SPA served by nginx | 8090 | — | React (Expo) |

- **Communication:** sync via REST through Kong; async via Kafka domain events per `09-microservices/event-catalog.md` and `05-architecture/pattern-guide.md` (Saga + Outbox — partially implemented, see AT-006).
- **Data isolation:** each service owns its DB/schema; cross-context refs are public IDs without physical FK (see `06-data/models.md` §4.5).

## Status

**Under construction — MVP Horizon 1 (0-3 months):** Identity, Academic, Scheduling, Attendance core validated; Biometric facial enroll; Justification workflow; Notification in-app. See `03-product/vision.md` roadmap.

## General & Specific Objectives

**General:** Develop a software solution for attendance recording, history consultation, justification management and report generation, ensuring security (Law 1581), usability and traceability.

**Specific (mapped to RF):**
- RF 1/1.1/1.2: Manage users/roles + CSV registration — Identity/Authorization
- RF 2/2.1/2.3/8: Environments, cohorts, courses, periods — Academic + Scheduling
- RF 3/3.1/3.1.1/3.1.2: Entry/exit via FACIAL/MANUAL/IMPORT — Attendance + Biometric
- RF 4/4.1/4.6: System configuration + alerts + role changes — Configuration + Notification
- RF 5/5.2: Histories (user change history now via infra logging after Audit retirement)
- RF 6: Justification workflow + supporting documents — Attendance
- RF 7: Reports/dashboards by date/cohort/environment — Attendance + Academic
- RNF 4: Security, Law 1581 consent, bcrypt cost 12 (ADR-008)
