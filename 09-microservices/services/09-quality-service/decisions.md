# Design Decisions — Quality Service

> Technical decisions specific to this service (not global architectural ones — those go in `05-architecture/decisions/`).
> Document here things that are not obvious: why this library, why this pattern, why this structure.

---

## DEC-001 PostgreSQL is the only persistence layer — a DB outage is a fatal boot error

**Date:** 2026-10-02
**Status:** Active (verified in `src/main.ts`, 2026-10-17)

**Context:**
Older docs describe this service as "in-memory + best-effort Postgres". The code does the opposite:
`start()` calls `connectDatabase()` and, on failure, logs `postgres unavailable — refusing to
start` and throws so Docker restarts the container.

**Decision:**
No in-memory fallback. The service is up only when Postgres is reachable; readiness
(`/health/ready`) reflects the probe result.

**Alternatives considered:**
- In-memory fallback with async sync-up: rejected — assessments are evidence; serving stale or
  volatile data would silently lose evaluations.

**Consequences:**
- ✅ No split-brain between what was answered and what was stored
- ⚠️ During a DB outage the endpoint is down (healthcheck flips to `not-ready`) instead of degrading

---

## DEC-002 Quality tooling lives in its own `quality` schema, shipped by the configuration changelog

**Date:** 2026-10-02
**Status:** Active

**Context:**
ISO 25010/29110/ISTQB assessments are not domain data of any bounded context, yet they need
durability, audit columns and migrations.

**Decision:**
They live in schema `quality` (7 tables + 5 enum types), created and migrated by
`FULL/database/07-ms-configuration-db/` alongside `configuration`. All SQL is qualified
`quality.<table>`; the pool sets `search_path` from `DB_SCHEMA` → `PG_SCHEMA` → `quality`,
whitelisted with an identifier regex before interpolation.

**Alternatives considered:**
- A DBML/bounded context of its own: rejected — quality tooling has no cross-context references
  and no owners outside the platform team.
- Mixed into the `configuration` schema: rejected — keeps query semantics clear and lets the
  quality tables be dropped as a unit.

**Consequences:**
- ✅ Tooling can evolve without touching the domain model or the DBML authority
- ⚠️ The `quality` tables are invisible in `database/faceattend_edu_mr_v4.dbml` — count them
  separately (`06-data/models.md` "Running database vs DBML" note: 36 tables total)

---

## DEC-003 The "event publisher" is a log hook, not messaging

**Date:** 2026-10-17
**Status:** Active

**Context:**
`src/infrastructure/messaging/event.publisher.ts` looks like a Kafka integration (and its comment
says "mismo patrón que academic/configuration"), but the file only adds a Fastify `onResponse`
log hook for `/api/v1/quality/` mutations.

**Decision:**
Keep the service free of any broker dependency: "domain events" are structured log lines the
platform can later ship to a log pipeline. Nothing in the service imports a Kafka client.

**Alternatives considered:**
- Real publish on assessment changes: rejected for now — there is no consumer anywhere in the
  platform (AT-006), so publishing would only add an unused dependency.

**Consequences:**
- ✅ Zero messaging failure modes; the service starts without Kafka
- ⚠️ The filename is misleading — read `events.md` before assuming quality emits platform events
