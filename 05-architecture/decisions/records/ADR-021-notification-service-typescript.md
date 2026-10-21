# ADR-021 — Notification Service rewritten in TypeScript/Fastify

- **ID:** ADR-021
- **Date:** 2026-10-18
- **Status:** Proposed
- **Authors:** FaceAttend EDU Team

---

## Context

`08-ms-notification` was the only Go service in the platform (Go 1.22 + Gin + pgx + zap),
which made the codebase four runtimes for nine services. The service itself is small — two
tables (`alert_type`, `alert`) with CRUD, an internal mail endpoint, an optional webhook and one
Kafka consumer — so the fourth runtime bought little and cost a dedicated toolchain.

The tech debt `AT-003` in `05-architecture/overview.md` §8 recorded exactly this: "notification
service Go runtime adds a 4th runtime for 2 tables — ops overhead", with the recommendation to
"evaluate consolidating to TS/Fastify if Go learning cost > benefit". Three other services
(03 academic, 07 configuration, 09 quality) already run the same TypeScript/Fastify hexagonal
layout, so the consolidation target was not a new pattern but an existing, proven one.

**Known constraints:**
- Functional parity only: every endpoint, validation rule and behaviour of the Go service had to
  survive the rewrite. No new features.
- The service must keep working with the same database schema (PostgreSQL, schema `notification`)
  with no migration.
- The Kafka consumer's at-least-once semantics and idempotency had to be preserved exactly —
  it is the only real event flow in the platform.

---

## Decision

**We decided:** rewrite `08-ms-notification` in TypeScript + Fastify + Zod + Drizzle + pino,
following the same hexagonal structure as `03-ms-academic`, and delete the Go implementation
entirely rather than run both side by side.

**Justification:**
The tiebreaker was consistency with the majority pattern, not raw performance. The service is
I/O-bound (Postgres, SMTP, Kafka) with trivial business logic, so Go's runtime advantages were
never on the critical path, while sharing the Fastify stack meant the auth guard, error envelope,
audit-field handling and test tooling became the same code family as 03/07/09 — a fix to one
propagates across four services instead of living in a fifth, unshared implementation.

---

## Evaluated alternatives

| Alternative | Pros | Cons | Reason for discarding |
|------------|------|------|-----------------------|
| **TypeScript + Fastify + Drizzle** (chosen) | Matches 03/07/09 exactly; one auth guard (`registerAuthGuard`), one error contract, one test runner (jest); Node already required by 3 services and both frontends; kafkajs is already a dependency elsewhere | Adds a fourth consumer of the same Postgres schema; Node's memory footprint is higher than Go's | — (chosen) |
| Keep Go/Gin | No migration work; tiny binary and low memory | Fourth runtime with no second Go service to share lint, CI, on-call knowledge or reviewers with; keeps `AT-003` open indefinitely | The overhead it was meant to avoid never paid off — the service stayed at two tables |
| Node + Express | Ubiquitous, huge ecosystem | Diverges from the Fastify pattern the other three TS services use: two HTTP frameworks, two error shapes, two guard wirings | Would replace one duplicated implementation with a different duplication |
| Merge into `07-ms-configuration` | One fewer container; both are small support services | Couples two bounded contexts that have independent schemas and change rates; configuration has no notification responsibility in the domain model | Different bounded contexts — merging would blur ownership, not reduce runtime count |

---

## Consequences

**Positive:**
- Back to **3 runtimes** platform-wide (Java, TypeScript, Python); `AT-003` closed.
- The auth guard, error envelope and audit-field handling are now shared across 03/07/08/09.
- Consistent structured logging (`pino`) across all TypeScript services; the `zap` logger left
  with the Go code.
- CI now covers the service through the existing `node-services` matrix (Node 20 → `npm ci`,
  `npm test`, `npm run build`) — it never had a dedicated Go job to begin with.

**Negative / Trade-offs:**
- Higher baseline memory per instance than the Go binary, irrelevant at this service's load but
  no longer a "lightweight sidecar".
- The Kafka consumer had to be re-expressed: kafkajs has no manual fetch/commit primitive like
  the Go reader's `FetchMessage`/`CommitMessages`, so the per-message retry-then-commit loop is
  ported *inside* `eachMessage` — the partition still blocks on a stuck message, but the
  mechanism is less obvious to a future maintainer.
- The consumer still has **no DLQ**; a message that keeps failing blocks its partition until it
  is classified non-retryable or intervened on manually.

**Impact on the system:**
- Affected services: `08-ms-notification` (rewritten in place), and `ci.yml`'s `node-services`
  matrix (now four entries instead of three).
- Documents that must be updated: `07-api/*`, `09-microservices/service-catalog.md`,
  `09-microservices/services/08-notification-service/*`, `10-devops/*`, `11-quality/*`,
  `13-operations/*`, `_stacks/README.md`, ADR-005/ADR-012/ADR-018 implementation notes,
  `05-architecture/overview.md` (AT-003, AT-004).
- Database: **unchanged** — same schema, same migrations, same `DATABASE_URL`.

---

## Risks

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|-----------|
| Behaviour drift between the deleted Go service and the TS port | Medium | Medium | Spec pinned the Go behaviour first (`2026-10-18-notification-service-typescript-design.md` §2); parity is covered by 16 jest test files including an `authGuard` integration test and a consumer test |
| **Missing alert-type seeds make the consumer block its partition** | High | High | The handler requires `JUSTIFICATION_APPROVED` / `JUSTIFICATION_REJECTED` rows that `001-seed-alert-type-table.yaml` does not insert; without them every event retries forever. Needs a new incremental changeset (existing seeds are never edited) — **open, not resolved by this ADR** |
| Poison message blocks the `attendance.justification.events` partition | Low | High | Non-retryable events (bad status, zero actor id, empty `reviewedAt`, unparseable JSON) are logged and skipped so the offset advances; a DLQ remains open debt (`AT-006`) |
| Silent loss of alerts when `KAFKA_BROKERS` is unset | Medium | Medium | The consumer is deliberately opt-in; the absence must be visible in deployment config, not discovered in an incident |
| `npm audit` findings in the new dependency tree | High | Low | Already an informational (`continue-on-error`) step in CI; Dependabot tracks them weekly |

---

## References

- Design spec: `FULL/docs/superpowers/specs/2026-10-18-notification-service-typescript-design.md`
- Service source: `FULL/back-end/08-ms-notification/` (`src/main.ts`,
  `src/infrastructure/http/routes.ts`, `src/infrastructure/http/authGuard.ts`,
  `src/infrastructure/messaging/justification.consumer.ts`)
- Supersedes in part: ADR-005 (technology stack — the Go row for 08)
- Related to: ADR-004 (attendance consistency — the consumer's topic), ADR-006 (message broker),
  ADR-012 (observability — logger change), ADR-018 (CI — job matrix)
- Service-level record: `09-microservices/services/08-notification-service/decisions.md` DEC-004
