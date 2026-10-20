# Communication Patterns

> **Corrected 2026-10-11, re-verified against `FULL/`.** The previous version of this file
> described a target architecture that was never built as written: it named an "Audit Service"
> that was removed from the domain (see `refactor/remove-audit-domain`), and it described
> Kafka as a working publish/subscribe mechanism between services. Neither is true today. The
> real system is almost entirely synchronous HTTP; Kafka exists and has **exactly one consumer**
> (`08-ms-notification` reading `attendance.justification.events`) — every other topic is
> published into and never read from. Sections below
> are marked **(real)** or **(target / not implemented)** throughout.

---

## 1. Synchronous communication (real)

This is how the system actually communicates. Every business operation — reads and writes alike
— is a direct HTTP call, not an event.

| Aspect | Detail |
|--------|--------|
| **Protocol** | HTTP/REST over TLS (TLS terminates at Kong in prod) |
| **Entry point** | API Gateway — Kong OSS 3.6, port 8000 |
| **Consumer** | Web, Mobile, and any direct caller of a service's own port |
| **Pattern** | Request → Response, always synchronous |
| **Use cases** | Everything: queries, every command, login, every CRUD operation, biometric match, attendance write |

### Real request flow (every authenticated call)

```
Client ──▶ Kong :8000
             │  edge pre-function: GET ms-identity/api/v1/sessions/{id}
             │  (401/503 here if the session is missing/invalid — the call
             │   never reaches the service)
             ▼
          Domain Service :808x
             │  its own guard re-validates: GET ms-identity/api/v1/sessions/{id}
             │  + GET ms-authorization/api/v1/auth/evaluate (when the route
             │  needs a permission) — see 07-api/authentication.md §1
             ▼
          Its own database (Postgres schema or MongoDB)
             │
Client ◀── Kong ◀────────────┘
```

Every one of the 9 services runs this same two-hop auth chain on its own — not just the request
the client made, but two more synchronous calls to `ms-identity` and (conditionally)
`ms-authorization` hidden inside it. See `08-uml/diagrams/source/seq-login.mmd` (SEQ-02) for the
full sequence.

### The only two real inter-service calls beyond the auth chain

| Caller | Callee | Path | Why |
|--------|--------|------|-----|
| `01-ms-identity` | `08-ms-notification` | `POST /api/v1/emails` (internal network only, **not** routed through Kong) | Delivers the password-recovery code after `POST /auth/forgot-password` |
| `08-ms-notification` | External system (`WEBHOOK_URL`, optional) | `POST <WEBHOOK_URL>` (HMAC-SHA256 signed when `WEBHOOK_SECRET` is set) | Fires once per `POST /api/v1/alerts`, only when `WEBHOOK_URL` is configured — skipped silently otherwise. Not a call between FaceAttend services, an outbound integration point |

No other service calls another service's business API. In particular: `03-ms-academic` and
`04-ms-scheduling` never call each other (a `schedule_block`'s `cohort_id`/`course_id`/
`instructor_actor_id` are never checked against Academic before being saved — see
`09-microservices/communication-patterns.md` §3 below), and `05-ms-attendance` never calls
`06-ms-biometric` (the biometric match happens client-side before the attendance write; see
`08-uml/diagrams/source/seq-facial-attendance.mmd`, SEQ-01).

---

## 2. Asynchronous communication (one real consumer, rest target state)

| Aspect | Detail |
|--------|--------|
| **Transport** | Kafka (KRaft, `:9092`), ADR-006 |
| **Pattern (real)** | Publish for six services; one consumer (`08-ms-notification` on `attendance.justification.events`) |
| **Pattern (target)** | Publish → Subscribe, eventual consistency |
| **Use cases today** | Justification reviewed → dashboard alert. All other events are produced and never read |

### What's real

Six services publish best-effort (failures are only logged, no retry, no outbox, no DLQ — AT-006):

| Producer | Topics |
|----------|--------|
| `01-ms-identity` | `identity-events`, `password-policy-events` (**`city-events` no longer published** — the `city` table was dropped) |
| `02-ms-authorization` | `role-events`, `permission-events` |
| `03-ms-academic` | `academic-events` |
| `04-ms-scheduling` | `class-session-events`, `environment-events`, `schedule-block-events` |
| `05-ms-attendance` | `attendance-events`, `justification-events`, `justification-type-events`, `document-events`, `attendance.justification.events` (review flow) |
| `07-ms-configuration` | `configuration-events` |

`06-ms-biometric`, `08-ms-notification` and `09-ms-quality` publish nothing. **One service consumes a
topic:** `08-ms-notification` (kafkajs, consumer group `notification-service`, started only when
`KAFKA_BROKERS` is set) reads `attendance.justification.events` — published by
`ReviewJustificationUseCaseImpl` after commit — and raises the `JUSTIFICATION_APPROVED` /
`JUSTIFICATION_REJECTED` dashboard alert (ERF 6.3). Everything else is still unread:
an `attendance-events` message is published on every attendance write,
but nothing raises a `notification.alert` from it — those alerts only exist when something calls
`POST /api/v1/alerts` directly. See `08-uml/diagrams/source/seq-facial-attendance.mmd` (SEQ-01)
for the attendance write's own notes on this gap.

### What was planned but never built

- A consumer side for any of the topics **other than** `attendance.justification.events`.
- Dead Letter Queues per service (the table in the old version of this file, `identity.dlq`,
  `attendance.dlq`, etc., does not exist — there is nowhere to route a failure to: the one real
  consumer blocks its partition and retries in place instead of parking the message).
- Retry-with-backoff for event **delivery** (publish side is still fire-and-forget; the backoff
  that exists lives inside the notification consumer's handler loop).
- An `audit.dlq` / Audit Service consuming anything — **the Audit bounded context was removed**
  (see `05-architecture/overview.md` and `02-domain/entities-and-rules.md`'s note on RN-30/31/
  37–41/44/45 being retired). Do not reintroduce "Audit Service" in this file or any diagram.

---

## 3. External ID references (real)

Services never use direct database joins across boundaries — this part of the original design
held. But "the referencing service validates the ID before saving" did **not**: in practice, no
service calls another to check a cross-context ID exists before writing it.

| Pattern | Description | Real example |
|---------|-------------|---------------|
| Public ID reference | Store the UUID/ID of an entity owned by another context, no FK | `attendance.attendance_record.academic_actor_id` → `academic.academic_actor` (AGENTS.md §3) |
| **No existence check (real, not a documented exception)** | The write succeeds even if the referenced id does not exist anywhere | `scheduling.schedule_block.cohort_id`/`course_id`/`instructor_actor_id` are saved with no call to `03-ms-academic` to confirm they exist |
| Local projection (target, not observed) | Cache a materialized copy of cross-context data for read performance | Not implemented in any service's persistence layer |

### Rules (still true)

1. Cross-context references are UUIDs or the referenced table's own PK — never resolved through
   a cross-schema FK (AGENTS.md §2.1).
2. No service may issue SQL joins against another service's database (true: schema-per-context,
   separate connection pools, separate Mongo database for Biometric).
3. **Not enforced today:** "referencing services must handle the case where the referenced entity
   does not yet exist" — there is no existence check to begin with, so there is nothing that
   fails fast; a dangling reference is only caught later, if at all, by a human reading a report.

---

## 4. Communication matrix (real)

| Source | Target | Channel | Pattern | Purpose |
|--------|--------|---------|---------|---------|
| Web / Mobile | Kong | HTTP/REST | Sync | Every business call |
| Kong | Every one of the 9 services | HTTP/REST | Sync | Routing + edge session check |
| Every one of the 9 services | `01-ms-identity` | HTTP/REST | Sync | Re-validate the session (`GET /sessions/{id}`) on every authenticated call |
| 8 of the 9 services (all but `01-ms-identity` itself) | `02-ms-authorization` | HTTP/REST | Sync | Evaluate a permission (`GET /auth/evaluate`) when the route needs one |
| `01-ms-identity` | `08-ms-notification` | HTTP/REST (internal, no Kong) | Sync | Password-recovery e-mail |
| `08-ms-notification` | External `WEBHOOK_URL` (optional) | HTTP/REST | Sync | Alert fan-out, only when configured |
| `01/02/03/04/05/07`-ms-* | Kafka | Producer | Fire-and-forget | Published, never consumed — see §2 |

There is no "Identity → Academic (Person creation)", no "Scheduling → Attendance (class session
lifecycle)", no "Biometric → Attendance (verification)", no "Configuration → Identity/Academic"
event flow, and no Audit Service consuming anything. If a future diagram or doc proposes one of
these, it is describing target state (ADR-006's eventual-consistency goal), not what runs today
— label it as such.

---

## 5. Error handling patterns

### Synchronous errors (real)

| HTTP Code | Meaning | Action |
|-----------|---------|--------|
| 400 | Bad request / validation error | Return error details to caller |
| 401 | Unauthorized | Missing/invalid/expired session — caller must re-authenticate |
| 403 | Forbidden | Caller lacks the required permission |
| 404 | Not found | Resource does not exist in the owning context |
| 409 | Conflict | Business rule violation (e.g., duplicate attendance record) |
| 422 | Unprocessable | Valid JSON but semantically incorrect |
| 429 | Rate limited | Kong's redis rate-limiting plugin tripped — caller must back off |
| 503 | Service unavailable | Kong's edge check or a service's own guard could not reach identity/authorization |
| 5xx | Server error | No circuit breaker exists (tech debt); retry with backoff is a client-side responsibility only |

### Asynchronous errors (partially implemented)

| Mechanism | Status |
|-----------|--------|
| Dead Letter Queue (DLQ) | Not implemented — the one real consumer blocks its partition and retries in place, so there is nowhere to fail into |
| Retry with backoff | Publish side: not implemented, a failed Kafka publish is only logged (AT-006). Consume side: implemented inside `08-ms-notification`'s handler (exponential backoff capped at 30 s; non-retryable events are skipped and the offset committed) |
| Idempotent handlers | Implemented for the one real handler (`JUSTIFICATION_APPROVED`/`REJECTED` alerts are deduplicated on actor + alert type + `raised_at`); N/A for the rest |
| Alerting on DLQ depth | N/A |
| Per-service DLQ topics | Not implemented — remove any reference to `*.dlq` topic names from other docs; they describe a design that was never built |

---

## 6. Interaction style summary

| Style | When (real) | Latency | Consistency | Complexity |
|-------|------|---------|-------------|------------|
| Sync (HTTP/REST) | Everything — every query and every command, plus the edge + per-service auth chain on every call | Low (ms), but every authenticated request costs at least 2–3 extra internal hops (session + permission checks) | Strong (request scope) | Low per call, but duplicated auth logic across 4 stacks (AT-004) |
| Async (Kafka) | Nothing consumes it yet | N/A — events are produced and then ignored | N/A | Low (producer-only), but misleading if read as "the system reacts to its own events" |

---

## Correlations

- Service catalog → `09-microservices/service-catalog.md`
- Event catalog (producers only) → `09-microservices/event-catalog.md`
- OpenAPI contracts → `07-api/contracts/openapi/`
- Domain events (intended) vs. what's wired → `02-domain/domain-events.md`
- Auth chain detail → `07-api/authentication.md`
- Sequence diagrams for the flows that are actually cross-service → `08-uml/diagram-index.md`
  (SEQ-01 facial attendance, SEQ-02 login, SEQ-03 justification approval, SEQ-04 biometric update
  case duplication, SEQ-05 password recovery)
