# Design Decisions — Notification Service

> Technical decisions specific to this service (not global architectural ones — those go in `05-architecture/decisions/`).
> Document here things that are not obvious: why this library, why this pattern, why this structure.

---

## DEC-001 The service is a REST CRUD + mail dispatcher, not a Kafka consumer

**Date:** 2026-10-11
**Status:** Superseded 2026-10-18 by DEC-004 / ADR-021 — the TypeScript rewrite added the
platform's first Kafka consumer (`attendance.justification.events`). Kept for history.

**Context:**
`SERVICE.md` §5 documents Kafka consumption (`attendance-events`, `scheduling-events`, …) to raise
absenteeism/tardiness alerts. The Go code contains **no Kafka client at all**.

**Decision:**
Alerts are rows created through `POST /api/v1/alerts` by the client/orchestrating side; email is
sent through the internal `POST /api/v1/emails`. The service deliberately has no broker dependency,
so it starts, scales and fails independently of Kafka.

**Alternatives considered:**
- Subscribing to domain topics to derive alerts: rejected for now — no consumer existed anywhere in
  the platform (AT-006), and alerting rules would need replay/backfill semantics first.

**Consequences:**
- ✅ No Kafka in the container: fewer moving parts, trivially horizontal-scalable
- ⚠️ Alert *derivation* (e.g. "3 absences → ABSENTEEISM") lives in clients, not here —
  `SERVICE.md` must be read as target, not as built behavior

> **What changed (2026-10-18):** the rejection above rested on "no consumer exists anywhere". The
> justification-review consumer made that premise obsolete. The consumer is **opt-in** — it only
> starts when `KAFKA_BROKERS` is set — so the "no broker dependency by default" property of this
> decision still holds for environments that do not set it. Absenteeism/tardiness derivation is
> still not implemented and remains out of scope.

---

## DEC-002 Email is gated by configuration and answers 202 instead of failing

**Date:** 2026-10-11
**Status:** Active

**Context:**
Password-reset/verification mails must never break the issuing flow, and dev/CI environments have
no SMTP server.

**Decision:**
`POST /api/v1/emails` uses nodemailer (rewritten from the Go stdlib `net/smtp` in 2026-10-18); if
`SMTP_HOST` is empty it returns
`202 {"status":"skipped","reason":"smtp_not_configured"}` (fail-open at the transport level).
Internal callers may be authenticated with `X-Internal-Token` (`EMAIL_INTERNAL_TOKEN`).

**Alternatives considered:**
- Hard-failing when SMTP is down: rejected — it would turn an email problem into an identity
  (login/reset) outage.

**Consequences:**
- ✅ Identity flows are never blocked by mail configuration
- ⚠️ A misconfigured production `SMTP_HOST` silently drops mail (only visible as `skipped`)

---

## DEC-003 Alerts reference subjects by public ID, never by FK

**Date:** 2026-09-12
**Status:** Active

**Context:**
`alert.academic_actor_id` points at Academic rows (another bounded context/schema).

**Decision:**
Store the raw `academic_actor_id` with **no cross-schema FK** (rule in `AGENTS.md` §2.1); resolve
names/details through the API when rendering the dashboard.

**Alternatives considered:**
- Cross-schema FK or a denormalized actor snapshot: rejected — couples schemas and duplicates data
  Academic already owns.

**Consequences:**
- ✅ Notification migrates independently of Academic
- ⚠️ Deleting/anonymizing an actor can leave alerts whose subject no longer resolves — dashboards
  must tolerate dangling IDs

---

## DEC-004 Rewritten in TypeScript/Fastify, keeping the Go handler semantics

**Date:** 2026-10-18
**Status:** Active

**Context:**
The Go implementation made this the platform's only fourth runtime for two CRUD tables
(`AT-003`), and 03/07/09 already proved the TS/Fastify hexagonal pattern in this codebase. The
rewrite was also the opportunity to finally give the service its Kafka consumer.

**Decision:**
Rewrite the service in TypeScript + Fastify + Zod + Drizzle + pino, porting each Go handler
one-for-one, and add the `attendance.justification.events` consumer with kafkajs
(group `notification-service`, opt-in via `KAFKA_BROKERS`). Auth moves from the Go
`AuthMiddleware` to the same `registerAuthGuard` used by 03/07/09. Rationale and the full
options matrix are recorded in `05-architecture/decisions/records/ADR-021-notification-service-typescript.md`.

**Alternatives considered:**
- Keep Go: rejected — a fourth runtime for two tables, with no Go anywhere else in the platform
  to share tooling, linting or on-call knowledge with.
- Node/Express: rejected — the other TS services use Fastify, and matching it means one guard,
  one error contract and one set of conventions instead of two.

**Consequences:**
- ✅ Back to **3 runtimes** platform-wide; `AT-003` closed
- ✅ The auth guard, error envelope and audit-field handling are now the same code family as
  03/07/09, so a fix propagates across four services instead of one
- ✅ First real end-to-end event flow: justification reviewed → dashboard alert
- ⚠️ The consumer has no DLQ — a poison message blocks its partition until it is skipped or the
  offset is intervened on manually
- ⚠️ kafkajs has no manual commit primitive; the Go reader's per-message retry loop had to be
  ported *inside* `eachMessage` to keep the same "block the partition rather than drop the
  event" behaviour
