# Design Decisions — Notification Service

> Technical decisions specific to this service (not global architectural ones — those go in `05-architecture/decisions/`).
> Document here things that are not obvious: why this library, why this pattern, why this structure.

---

## DEC-001 The service is a REST CRUD + mail dispatcher, not a Kafka consumer

**Date:** 2026-10-11
**Status:** Active (verified 2026-10-17)

**Context:**
`SERVICE.md` §5 documents Kafka consumption (`attendance-events`, `scheduling-events`, …) to raise
absenteeism/tardiness alerts. The Go code contains **no Kafka client at all**.

**Decision:**
Alerts are rows created through `POST /api/v1/alerts` by the client/orchestrating side; email is
sent through the internal `POST /api/v1/emails`. The service deliberately has no broker dependency,
so it starts, scales and fails independently of Kafka.

**Alternatives considered:**
- Subscribing to domain topics to derive alerts: rejected for now — no consumer exists anywhere in
  the platform (AT-006), and alerting rules would need replay/backfill semantics first.

**Consequences:**
- ✅ No Kafka in the container: fewer moving parts, trivially horizontal-scalable
- ⚠️ Alert *derivation* (e.g. "3 absences → ABSENTEEISM") lives in clients, not here —
  `SERVICE.md` must be read as target, not as built behavior

---

## DEC-002 Email is gated by configuration and answers 202 instead of failing

**Date:** 2026-10-11
**Status:** Active

**Context:**
Password-reset/verification mails must never break the issuing flow, and dev/CI environments have
no SMTP server.

**Decision:**
`POST /api/v1/emails` uses stdlib `net/smtp`; if `SMTP_HOST` is empty it returns
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
