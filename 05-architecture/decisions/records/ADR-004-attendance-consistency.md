# ADR-004 — Cross-Context Consistency in the Attendance Registration Flow

- **ID:** ADR-004
- **Date:** 2026-09-03
- **Status:** Proposed
- **Implementation note (2026-10-05, verified against `FULL/`):** **partially implemented.** Domain events are published straight to Kafka (`KafkaTemplate` in Java, `kafkajs` in TS) but there is **no transactional outbox table, no DLQ and no compensation handlers** — publish failures are swallowed (`.catch(() => undefined)`). Recorded as tech debt `AT-006` in `05-architecture/overview.md` §8.
- **Authors:** [Diego Andrés Gutiérrez Nuñez, Juan David Arboleda Perdomo, Jonattan Steven Rizo Solano]

---

## Context

The system's most critical use case — facial-scan attendance registration (SRS Use Case 1
"Registro asistencia" [Attendance Registration] and Use Case 2 "Escaneo facial" [Facial Scan]; RF 3.1, RF 3.1.1, RF 3.1.2) — spans at
least three bounded contexts in a single business operation:

1. **Biometric** — the captured face is compared against stored embeddings
   (`Facial_Embedding`) to identify the student. **Note:** Per ADR-003, Biometric is persisted
   in an external MongoDB instance, not in the shared PostgreSQL database.
2. **Attendance** — on a match, an `attendance_record` row is created, referencing an open
   `class_session` (`status = 'Open'`), with a uniqueness constraint
   `(class_session_id, academic_actor_id)`.
3. **Observability** — the action is emitted as a platform audit event for traceability, per
   `00-governance/security-policy.md`'s mandatory security/audit logging rules (no `audit` schema; Audit context removed).

In a classic microservices architecture, a business operation spanning multiple services with
separate databases requires a distributed consistency pattern — typically Saga with
compensating transactions, as documented generically in `05-architecture/pattern-guide.md`.
That pattern exists specifically to handle the case where an ACID transaction *cannot* span
the services involved, because each has its own database and its own transaction boundary.

ADR-002 establishes that most bounded contexts in this project live as schemas **inside a
single PostgreSQL instance**. However, ADR-003 carves out an exception: **Biometric is
persisted in MongoDB**, not PostgreSQL. This means the physical constraint that Saga is
designed to work around partially applies: Biometric (MongoDB) cannot participate in a
PostgreSQL ACID transaction. Attendance uses a local transaction in its own schema; audit is a platform event outside the transaction.

This means the team faces a genuine, non-obvious choice that does not reduce to "just use
Saga because microservices" nor to "just use one big transaction because it's easy" — each
option has real trade-offs given where this system is headed.

**Known constraints:**
- `class_session` has a uniqueness constraint on `(schedule_block_id, session_date)`, and `attendance_record` has
  one on `(class_session_id, academic_actor_id)` — both must be respected atomically to avoid duplicate
  registrations under concurrent scans.
- `00-governance/security-policy.md` requires certain actions to always emit a platform audit event,
  which must not be silently lost (delivery is best-effort, outside the DB transaction).
- ADR-002 preserves the possibility of extracting a context (e.g., Attendance, if it becomes
  the highest-traffic module) into an independent service later.
- Biometric data lives in MongoDB (ADR-003), so any cross-database consistency between
  Biometric reads and Attendance/Audit writes must be handled at the application layer.

---

## Decision

**We decided:** The attendance registration flow uses a **single local ACID transaction**
in the `attendance` schema for the write path (create `attendance_record` row).
The platform audit event is emitted **after commit** (best-effort, not part of the DB transaction).
The `Biometric` match (read-only comparison against `facial_embeddings` in MongoDB) happens
**before** this transaction starts and is treated as a precondition, not as a participant in
the write transaction — a failed or inconclusive match simply never reaches the transactional
write step.

Any action that is genuinely decoupled from the immediate request/response cycle — for
example, sending an attendance-anomaly notification to a guardian or instructor (RF 4.1,
RF 6.3) — is **not** part of this transaction. Those reactions are triggered via an
**in-process domain event** published after the transaction commits (an internal event bus
within the single deployable process — not a message broker), consumed by the
`Configuration`/notification-handling code asynchronously within the same process.

**Justification:** Because Attendance lives in a single PostgreSQL schema
(per ADR-002), using a local ACID transaction for the write path is strictly simpler and
strictly more correct than Saga: it gives real atomicity and consistency guarantees with no
compensating-transaction logic to write, test, or maintain. Audit is platform observability (no `audit` schema since Audit removal), so no `Attendance` row ever waits on an audit-table write. Reaching for Saga here
would be solving a problem this project's current architecture does not have for the
Attendance write.

The Biometric context (ADR-003) is intentionally excluded from this transaction because it
lives in MongoDB, not PostgreSQL. The read from Biometric is treated as an external service
call that provides a precondition (match/no-match) before the transactional write begins.
This is consistent with the general principle that cross-database operations cannot share
ACID transactions.

At the same time, reserving in-process events (not a full message broker) for genuinely
decoupled side effects (notifications) keeps that logic loosely coupled and easy to migrate to
a real broker later, without forcing today's core write path through unnecessary
asynchronous machinery.

This decision is explicitly tied to ADR-002 and ADR-003: **if `Attendance`
is ever extracted into an independently deployed service, this ADR must be revisited**,
because the local-transaction guarantee disappears the moment the schemas involved no longer
share a database connection — at that point, the Saga/Outbox patterns already documented in
`pattern-guide.md` become the correct tool, and this ADR should be marked "Superseded".

---

## Evaluated alternatives

| Alternative | Pros | Cons | Reason for discarding |
|------------|------|------|-----------------------|
| **Local ACID transaction (attendance) + in-process event for decoupled side effects (chosen)** | True atomicity with no compensating logic; simplest correct option given ADR-002 and ADR-003; audit is a post-commit platform event, not a DB write; decoupled effects (notifications) stay loosely coupled | Only valid as long as Attendance stays in one PostgreSQL schema; must be revisited if Attendance is later extracted; Biometric (MongoDB) cannot participate in this transaction | — (chosen) |
| Choreographed Saga with compensating transactions across all three contexts | Matches the "textbook" microservices consistency pattern already documented in `pattern-guide.md`; would already be correct if contexts were extracted to separate services | Solves a distributed-transaction problem that does not exist today for Attendance, since it lives in one PostgreSQL schema; adds compensating-transaction code, event schemas, and failure-handling logic with no present benefit; significantly more code and testing surface for the team to maintain | Premature — it optimizes for a deployment topology (separate databases) that ADR-002 explicitly does not adopt right now for Attendance |
| One large transaction covering Biometric read + Attendance write + notification dispatch | Conceptually the "simplest" single unit | Biometric is in MongoDB (ADR-003) and cannot participate in a PostgreSQL ACID transaction; notification dispatch is an I/O-bound, potentially slow, and non-critical-path operation (e.g., sending an email/SMS); including it inside the same DB transaction would hold locks longer than necessary and couple an unrelated concern (messaging) to the attendance write's success/failure | Biometric's MongoDB location makes this physically impossible; mixing a fast, critical, transactional concern (attendance) with a slow, non-transactional one (notifications) unnecessarily increases lock duration and failure coupling |

---

## Consequences

**Positive:**
- No duplicate `attendance_record` rows are possible — the database
  guarantees it via `uq_attendance_session_actor`.
- No compensating-transaction code to write or maintain for the core write path.
- Notification, platform audit events and other side effects remain decoupled via in-process events, keeping the
  core write path fast and focused.
- Clear separation between Biometric (MongoDB, read-only precondition) and the transactional
  write path (PostgreSQL, ACID guarantee).

**Negative / Trade-offs:**
- This design is coupled to ADR-002's premise (single PostgreSQL instance for Attendance)
  and ADR-003's premise (Biometric in MongoDB). It is **not** a general-purpose
  distributed-systems solution and will need explicit revisiting — not silent reuse — if
  Attendance is extracted to its own service.
- The in-process event mechanism is not durable across process restarts unless explicitly
  backed by something like the Outbox pattern; a crash between commit and event dispatch could
  drop a notification or platform audit event. This is an accepted risk for notifications/audit events (non-critical for write integrity), which is why only the attendance write itself uses a
  transaction rather than an event.
- Cross-database consistency between Biometric (MongoDB) and Attendance (PostgreSQL)
  is eventual, not ACID — this is acceptable because Biometric reads are precondition checks
  (match/no-match) that happen before the transactional write, and the system can tolerate
  the small window of inconsistency.

**Impact on the system:**
- Affected services: `Biometric` (read path, MongoDB), `Attendance` (PostgreSQL),
  `Notification` (event consumer). Audit is platform observability, not a service.
- Documents that must be updated:
  - `05-architecture/pattern-guide.md` — "Patterns adopted in this project" table should mark
    Saga as "No — see ADR-004; local ACID transaction used instead, given ADR-002's
    single-instance strategy" and note in-process events as the mechanism for decoupled side
    effects.
  - `02-domain/domain-events.md` — should document the in-process event(s) emitted after the
    attendance write commits (e.g., `AttendanceRegistered`), including that delivery is
    in-process/best-effort, not broker-backed, unless upgraded later.
  - Sequence diagram in `08-uml/diagrams/source/` for the attendance registration flow, to
    make the transaction boundary and the event-publish point visually explicit for future
    contributors.

---

## Risks

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|-----------|
| A future decision extracts Attendance to its own service without revisiting this ADR | Medium (natural temptation once the system grows) | High (silently breaks the atomicity guarantee this ADR relies on) | Add an explicit cross-reference from ADR-002's "Consequences" section to this ADR, so any extraction discussion is forced to reconsider it |
| Process crash between transaction commit and in-process event dispatch drops a notification | Low-Medium | Low (notifications are not safety-critical; a missed one does not corrupt attendance data) | Accept as a known limitation; consider Outbox pattern (already documented in `pattern-guide.md`) if notification reliability requirements increase |
| Long-running transaction holds locks if the write path grows to include more logic later | Low today | Medium if it happens | Keep the transaction scoped strictly to Attendance writes; any new logic added to this flow must be evaluated against this boundary before being included inside the transaction |
| Cross-database inconsistency between Biometric (MongoDB) and Attendance (PostgreSQL) during the read-before-write window | Low (Biometric data is relatively stable; reads are fast) | Medium (could lead to false positive/negative matches if Biometric data changes during the window) | Treat Biometric read as a point-in-time snapshot; accept eventual consistency for this specific cross-database interaction; document the consistency model in `09-microservices/communication-patterns.md` |

---

## References

- Use Cases this flow implements → SRS "Registro asistencia" [Attendance Registration] (UC1) and "Escaneo facial" [Facial Scan] (UC2)
- Functional requirements → RF 3.1, RF 3.1.1, RF 3.1.2
- Table constraints → the system's entity-relationship model (`class_session`, `attendance_record`)
- Mandatory platform audit events → `00-governance/security-policy.md`
- Saga / Outbox pattern reference (for future extraction scenario) → `05-architecture/pattern-guide.md`
- Related to: ADR-002 (single-instance strategy this decision depends on for Attendance)
- Related to: ADR-003 (Biometric in MongoDB, excluded from PostgreSQL transaction)