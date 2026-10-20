# ADRs — Architecture Decision Records

ADRs document important architectural decisions. Each file = one decision.

## How to create an ADR

1. Copy `_template-adr.md`
2. Name it `ADR-NNN-short-title.md` (e.g.: `ADR-001-message-broker.md`)
3. Fill it in completely — especially the evaluated alternatives
4. Once accepted, the status is **permanent** (it is not deleted, it is "Superseded" by another ADR)

## Possible statuses

- `Proposed` — under discussion
- `Accepted` — approved by the team
- `Rejected` — evaluated and discarded (document why)
- `Superseded` — superseded by ADR-NNN (indicate which one)

## ADR register

| # | Title | Status | Date | Scope |
|---|-------|--------|------|-------|
| ADR-001 | Documentation language | Accepted | 2026-09-01 | Cross-cutting |
| ADR-002 | Modular monolith — schema-per-context | Accepted | 2026-09-03 | Architecture style |
| ADR-003 | Biometric persistence (MongoDB vectors) | Accepted | 2026-09-04 | Data |
| ADR-004 | Attendance consistency (Saga + Outbox) | Accepted | 2026-09-05 | Consistency |
| ADR-005 | Technology stack per service | Proposed | 2026-09-12 | Stack |
| ADR-006 | Message broker (Kafka) | Proposed | 2026-09-12 | Integration |
| ADR-007 | API Gateway (Kong OSS) | Accepted | 2026-09-12 | Edge |
| ADR-008 | Password hashing (bcrypt cost 12) | Accepted | 2026-09-12 | Security |
| ADR-009 | File storage | Accepted | 2026-09-13 | Infra |
| ADR-010 | Database migrations (Liquibase) | Accepted | 2026-09-13 | Data |
| ADR-011 | CI/CD platform (Jenkins in the ADR; GitHub Actions in practice) | Accepted | 2026-09-13 | DevOps |
| ADR-012 | Observability stack | Proposed | 2026-09-14 | Ops |
| ADR-013 | Frontend technology | Proposed | 2026-09-14 | UX |
| ADR-014 | Biometric SDK | Proposed | 2026-09-14 | Biometric |
| ADR-015 | Caching strategy | Proposed | 2026-09-14 | Perf |
| ADR-016 | Encryption at rest | Accepted | 2026-09-15 | Security |
| ADR-017 | Localization | Proposed | 2026-09-15 | UX |

> Update status to `Superseded by ADR-NNN` when replaced. Audit retirement (2026-09) is tracked in ADR-002/005/006 via `refactor/remove-audit-domain`.

## Implementation notes (verified 2026-10-05 against `FULL/`)

Decisions are unchanged; each ADR file carries its own `Implementation note` line under **Status**.

| ADR | What the code actually does |
|-----|------------------------------|
| ADR-002 | Schema-per-context + no cross-schema FK: live. Single deployable: superseded — 9 service containers + Kong + frontend |
| ADR-003 | MongoDB embeddings + `configuration.biometric_update_case`: implemented |
| ADR-004 | Events published to Kafka, but no outbox / DLQ / compensation (AT-006) |
| ADR-005 | Stack as decided (Java 21 + **Spring Boot 4.1**, TS Fastify, Python 3.12, Go 1.22) + ninth service `ms-quality` :8089 |
| ADR-006 | Kafka (KRaft `:9092`) is live with wired producers; no DLQ yet |
| ADR-007 | Kong 3.6 DB-less, global `pre-function` session check at the edge + cors + rate-limiting — **no `jwt` plugin**; opaque `sessionId` bearer, re-checked per MS |
| ADR-008 | `BCryptPasswordEncoder($2B, 12)` in `SecurityConfig` |
| ADR-010 | Liquibase 4.29 as 8 chained containers; YAML changesets (SQL inside); no Flyway |
| ADR-012 | Structured logs only — no Prometheus/Grafana/Jaeger deployed |
| ADR-015 | Not implemented — Redis only serves Kong rate-limiting |
