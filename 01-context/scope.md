# System Scope

> **Why this document exists:** Scope prevents scope creep and aligns expectations.
> It is equally important to define what the system does NOT do as what it does.
> Review this document at the start of each planning cycle.

---

## In Scope — MVP (Horizon 1, 0-3 months)

According to SRS + `03-product/vision.md` roadmap (validated in `02-domain` 8 contexts):

- RF 1/1.1/1.2: User management + CSV registration — Identity
- RF 1.3/1.4/1.5/1.6: Sign-in, password recovery/change, activate/deactivate — Identity
- RF 1.1.1/4.6 + 1.1.2: Role/permission management and role changes — Authorization
- RF 2/2.1/2.3/8: Environment, cohorts, programs, courses, periods, school management — Academic + Scheduling
- RF 3/3.1/3.1.1/3.1.2: Entry/exit records via FACIAL + MANUAL/IMPORT fallback — Attendance + Biometric
- RF 4/4.1/4.4: System and academic configuration (tardy tolerance, security params) — Configuration
- RF 6 + 6.1/6.2: Justification workflow + supporting documents + approval/rejection — Attendance
- RF 3/5/7: Attendance/absence history + reports by date range/cohort + analytical dashboard — Attendance

> **Post-Audit note (2026-09):** `User change history` (former RF 5.2/Audit) is now an **infra cross-cutting concern** (Kafka event log + centralized logging), not a business capability in `02-domain`. Not in MVP scope as product pillar.

## Out of Scope — MVP

- Teacher payroll, grading or payment processes.
- Complete physical access control (doors/turnstiles).
- Integration with external enrollment/grading/IdP systems (future candidate).
- Advanced internal messaging beyond in-app alerts (SMS/WhatsApp — future).
- Predictive dropout analytics / AI risk models.
- IoT device failure alerts — external supporting context (`_future-iot-service`) kept as **future hardware** integration, not in core 8 contexts (see `02-domain/domain-map.md` IoT note).
- Audit bounded context (removed 2026-09; now infra logging).
- Separate native mobile apps — MVP is **responsive web + mobile web** (React Native considered in Horizon 3, ADR-013).

## Candidates for future versions (Horizon 2-3)

- Fingerprint biometrics via `Biometric_Update_Case` workflow
- IoT classroom hardware integration (when devices available)
- Multi-school tenancy hardening, advanced analytics, gRPC internal path (see `05-architecture/overview.md` §9)

## Assumptions (to validate in pilot, see `03-product/problem-framing.md` §7 risks)

- Users have a modern browser + camera-capable device (mobile/web) and connectivity; fallback MANUAL/IMPORT available.
- Institution has at least one School/Program/Period seeded before cohort creation.
- Roles (Administrator/Instructor/Student) are pre-defined per `02-domain` Authorization.
- Initial users/environments can be seeded via CSV (RF 1.2).
- System supports educational management, not legal attendance enforcement — Law 1581 consent required for biometric (see ADR-017).
- Biometric vectors handled as PII: encryption at rest (ADR-016), consent workflow.

## Constraints

- SENA training project: 3 apprentices, no dedicated DevOps, 2-week sprints (`00-governance/agile-conventions.md`).
- Must align with SRS RF 1-8 + RNF 4 and Law 1581.
- Security/performance/portability per SRS §2; ports `8081`-`8089` (microservices) + `8000/8001` (Kong) + `8090` (frontend-web).
- Time/resources as academic project — scope frozen to MVP Horizon 1 until pilot validation.

## Correlations

- Problem framing & vision → `03-product/problem-framing.md`, `03-product/vision.md`
- Bounded contexts → `02-domain/domain-map.md` (8 contexts)
- Architecture style → `05-architecture/overview.md` / `ADR-002`
- Glossary source of truth → `01-context/glossary.md`
