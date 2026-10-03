# General Context — FaceAttend EDU

## Institutional Context

FaceAttend EDU is developed within the **Software Analysis and Development program — SENA**, as a technological solution to support attendance management in educational environments (initial pilot: one institution, 30+ learners per instructor). See `01-context/scope.md` for boundaries and `00-governance/agile-conventions.md` for team cadence.

## What is FaceAttend EDU?

**FaceAttend EDU** is a **web + mobile (responsive)** attendance-management platform that **automates entry/exit recording via facial recognition** with **manual/import fallback**, centralizes attendance histories, and provides **justification workflows, role-based reports and alerts**. It is a **modular monolith (schema-per-context, 8 bounded contexts) — Kafka-ready**, per `05-architecture/overview.md` and `ADR-002/006`.

## Problem it solves

In many academic processes, attendance depends on a single instructor taking manual lists. This causes **inconsistencies, administrative overload, impersonation risk, and delayed follow-up** on absences/late arrivals, especially with **30+ learners per cohort** (SRS §1.c justification).

## Main users

- **Student / Learner:** records attendance contactlessly, consults personal history, submits justifications.
- **Instructor / Teacher:** opens/closes Class_Sessions, views cohort attendance, receives late/absence alerts, reviews justifications.
- **Administrator:** manages users/roles/permissions, schools/programs/cohorts/environments, and system configuration (per-school and global).
- **System (future IoT):** external supporting actor for classroom hardware — out of MVP core (see `scope.md`).

## Technology stack (v1.1, 8 bounded contexts)

- **Backend:** Modular monolith — Java 21 + Spring Boot 3 (identity/authorization/scheduling/attendance), TypeScript + Fastify (academic/configuration), Python 3.12 + FastAPI (biometric), Go 1.22 + Gin (notification)
- **Databases:** Single logical DB `faceattend_db` on PostgreSQL 17 (7 schemas) + MongoDB 7 (facial/fingerprint vectors) + Redis 7 (cache/sessions)
- **Edge:** Kong OSS API Gateway :8080, JWT + RBAC, Kafka-ready events (ADR-006)
- **Infrastructure:** Docker Compose local, Flyway/Liquibase migrations (ADR-010), Jenkins CI/CD (ADR-011)

## Main architecture — Bounded Contexts & Services (Schema per Context, Single DB)

> **Ubiquitous Language:** service names below are the single source of truth — identical in `02-domain/domain-map.md`, `05-architecture/overview.md`, `09-microservices/service-catalog.md` and `06-data/models.md` (Informe 2026-09-22 §3.1-§3.2).

| # | Service | Bounded Context | Main responsibility | Port | DB | Technology |
|---|---|---|---|---|---|---|
| 01 | `identity-service` | Identity | Person, App_User, User_Session, Password_Policy, auth/JWT (City via external API) | 8081 | PostgreSQL | Java 21 + Spring Boot 3 |
| 02 | `authorization-service` | Authorization | Role, Permission, Role_Permission, User_Role (global roles) | 8083 | PostgreSQL | Java 21 + Spring Boot 3 |
| 03 | `academic-service` | Academic | School, Program, Academic_Period, Cohort, Course, Academic_Actor, Enrollment | 8084 | PostgreSQL | TypeScript + Fastify |
| 04 | `scheduling-service` | Scheduling | Environment, Schedule_Block, Class_Session (anti-double-booking) | 8087 | PostgreSQL | Java 21 + Spring Boot 3 |
| 05 | `attendance-service` | Attendance | Attendance_Record, Justification_Type, Justification, Supporting_Document, Attendance_Report (derived) | 8085 | PostgreSQL | Java 21 + Spring Boot 3 |
| 06 | `biometric-service` | Biometric | Facial_Embedding, Fingerprint_Embedding (MongoDB vectors) + Biometric_Update_Case (SQL, hosted in `configuration`) | 8086 | **MongoDB + SQL case** | Python 3.12 + FastAPI |
| 07 | `notification-service` | Notification | Alert_Type, Alert | 8090 | PostgreSQL | Go 1.22 + Gin |
| 08 | `configuration-service` | Configuration | Academic_Configuration, Security_Configuration | 8089 | PostgreSQL | TypeScript + Fastify |
| 99 | `api-gateway` | — (infrastructure) | Routing, auth, rate limiting | 8080 | Redis (cache) | Kong OSS |

- **Communication:** sync via REST through gateway; async via events (Kafka-ready) per `09-microservices/event-catalog.md` and `05-architecture/pattern-guide.md` (Saga + Outbox).
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
