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

## Technology stack (v1.1, 8 contexts)

- **Backend:** Modular monolith — Java 21 + Spring Boot 3 (identity/auth/scheduling/attendance), TypeScript + Fastify (academic/configuration), Python 3.12 + FastAPI (biometric), Go 1.22 + Gin (notification)
- **Databases:** PostgreSQL 17 (8 schemas, schema-per-context) + MongoDB 7 (facial/fingerprint vectors) + Redis 7 (cache/sessions)
- **Edge:** Kong OSS API Gateway :8080, JWT + RBAC, Kafka-ready events (ADR-006)
- **Infrastructure:** Docker Compose local, Flyway/Liquibase migrations (ADR-010), Jenkins CI/CD (ADR-011)

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
