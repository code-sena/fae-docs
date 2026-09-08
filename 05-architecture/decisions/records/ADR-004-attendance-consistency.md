# ADR-004 — Cross-Context Consistency in the Attendance Registration Flow

- **ID:** ADR-004
- **Date:** 2026-09-03
- **Status:** Proposed
- **Authors:** [Diego Andrés Gutiérrez Nuñez, Juan David Arboleda Perdomo, Jonattan Steven Rizo Solano]

---

## Context

The system's most critical use case — facial-scan attendance registration (SRS Use Case 1
"Registro asistencia" and Use Case 2 "Escaneo facial"; RF 3.1, RF 3.1.1, RF 3.1.2) — spans at
least three bounded contexts in a single business operation:

1. **Biometric** — the captured face is compared against stored embeddings
   (`Facial_Embedding`) to identify the student. **Note:** Per ADR-003, Biometric is persisted
   in an external MongoDB instance, not in the shared PostgreSQL database.
2. **Attendance** — on a match, an `Attendance` row is created, referencing an open
   `Class_Session` (`session_status = 'Open'`), with a uniqueness constraint
   `(student_id, session_id)`.
3. **Audit** — the action is expected to be recorded for traceability, per
   `00-governance/security-policy.md`'s mandatory security/audit logging rules.

In a classic microservices architecture, a business operation spanning multiple services with
separate databases requires a distributed consistency pattern — typically Saga with
compensating transactions, as documented generically in `05-architecture/pattern-guide.md`.
That pattern exists specifically to handle the case where an ACID transaction *cannot* span
the services involved, because each has its own database and its own transaction boundary.

ADR-002 establishes that most bounded contexts in this project live as schemas **inside a
single PostgreSQL instance**. However, ADR-003 carves out an exception: **Biometric is
persisted in MongoDB**, not PostgreSQL. This means the physical constraint that Saga is
designed to work around partially applies: Biometric (MongoDB) cannot participate in a
PostgreSQL ACID transaction. Attendance and Audit, however, share the same PostgreSQL instance
and can use a local transaction.

This means the team faces a genuine, non-obvious choice that does not reduce to "just use
Saga because microservices" nor to "just use one big transaction because it's easy" — each
option has real trade-offs given where this system is headed.

**Known constraints:**
- `Class_Session` has a uniqueness constraint on `(schedule_id, date)`, and `Attendance` has
  one on `(student_id, session_id)` — both must be respected atomically to avoid duplicate
  registrations under concurrent scans.
- `00-governance/security-policy.md` requires certain actions to always be logged for audit,
  which implies the audit write must not be silently lost.
- ADR-002 preserves the possibility of extracting a context (e.g., Attendance, if it becomes
  the highest-traffic module) into an independent service later.
- Biometric data lives in MongoDB (ADR-003), so any cross-database consistency between
  Biometric reads and Attendance/Audit writes must be handled at the application layer.

---

## Decision

**We decided:** The attendance registration flow uses a **single local ACID transaction**
across the `attendance` and `audit` schemas for the write path (create `Attendance` row +
write the corresponding `Audit_Event` row), issued from the `Attendance` module's use case.
The `Biometric` match (read-only comparison against `Facial_Embedding` in MongoDB) happens
**before** this transaction starts and is treated as a precondition, not as a participant in
the write transaction — a failed or inconclusive match simply never reaches the transactional
write step.

Any action that is genuinely decoupled from the immediate request/response cycle — for
example, sending an attendance-anomaly notification to a guardian or instructor (RF 4.1,
RF 6.3) — is **not** part of this transaction. Those reactions are triggered via an
**in-process domain event** published after the transaction commits (an internal event bus
within the single deployable process — not a message broker), consumed by the
`Configuration`/notification-handling code asynchronously within the same process.

**Justification:** Because Attendance and Audit share the same PostgreSQL database instance
(per ADR-002), using a local ACID transaction for the write path is strictly simpler and
strictly more correct than Saga: it gives real atomicity and consistency guarantees with no
compensating-transaction logic to write, test, or maintain, and no window where an
`Attendance` row could exist without its corresponding `Audit_Event`. Reaching for Saga here
— a pattern designed to simulate atomicity across systems that cannot share a transaction —
would be solving a problem this project's current architecture does not have for the
Attendance/Audit pair.

The Biometric context (ADR-003) is intentionally excluded from this transaction because it
lives in MongoDB, not PostgreSQL. The read from Biometric is treated as an external service
call that provides a precondition (match/no-match) before the transactional write begins.
This is consistent with the general principle that cross-database operations cannot share
ACID transactions.

At the same time, reserving in-process events (not a full message broker) for genuinely
decoupled side effects (notifications) keeps that logic loosely coupled and easy to migrate to
a real broker later, without forcing today's core write path through unnecessary
asynchronous machinery.

This decision is explicitly tied to ADR-002 and ADR-003: **if either `Attendance` or `Audit`
is ever extracted into an independently deployed service, this ADR must be revisited**,
because the local-transaction guarantee disappears the moment the schemas involved no longer
share a database connection — at that point, the Saga/Outbox patterns already documented in
`pattern-guide.md` become the correct tool, and this ADR should be marked "Superseded".

---

## Evaluated alternatives

| Alternative | Pros | Cons | Reason for discarding |
|------------|------|------|-----------------------|
| **Local ACID transaction (attendance + audit) + in-process event for decoupled side effects (chosen)** | True atomicity with no compensating logic; simplest correct option given ADR-002 and ADR-003; no risk of an Attendance row existing without its Audit entry; decoupled effects (notifications) stay loosely coupled | Only valid as long as Attendance and Audit share one database instance; must be revisited if either context is later extracted; Biometric (MongoDB) cannot participate in this transaction | — (chosen) |
| Choreographed Saga with compensating transactions across all three contexts | Matches the "textbook" microservices consistency pattern already documented in `pattern-guide.md`; would already be correct if contexts were extracted to separate services | Solves a distributed-transaction problem that does not exist today for Attendance/Audit, since both schemas are in one PostgreSQL instance; adds compensating-transaction code, event schemas, and failure-handling logic with no present benefit; significantly more code and testing surface for the team to maintain | Premature — it optimizes for a deployment topology (separate databases) that ADR-002 explicitly does not adopt right now for Attendance/Audit |
| One large transaction covering Biometric read + Attendance write + Audit write + notification dispatch | Conceptually the "simplest" single unit | Biometric is in MongoDB (ADR-003) and cannot participate in a PostgreSQL ACID transaction; notification dispatch is an I/O-bound, potentially slow, and non-critical-path operation (e.g., sending an email/SMS); including it inside the same DB transaction would hold locks longer than necessary and couple an unrelated concern (messaging) to the attendance write's success/failure | Biometric's MongoDB location makes this physically impossible; mixing a fast, critical, transactional concern (attendance + audit) with a slow, non-transactional one (notifications) unnecessarily increases lock duration and failure coupling |

---

## Consequences

**Positive:**
- No duplicate or inconsistent `Attendance`/`Audit_Event` pairs are possible — the database
  guarantees it.
- No compensating-transaction code to write or maintain for the core write path.
- Notification and other side effects remain decoupled via in-process events, keeping the
  core write path fast and focused.
- Clear separation between Biometric (MongoDB, read-only precondition) and the transactional
  write path (PostgreSQL, ACID guarantee).

**Negative / Trade-offs:**
- This design is coupled to ADR-002's premise (single PostgreSQL instance for Attendance and
  Audit) and ADR-003's premise (Biometric in MongoDB). It is **not** a general-purpose
  distributed-systems solution and will need explicit revisiting — not silent reuse — if any
  involved context is extracted to its own service.
- The in-process event mechanism is not durable across process restarts unless explicitly
  backed by something like the Outbox pattern; a crash between commit and event dispatch could
  drop a notification. This is an accepted risk for notifications (non-critical), but would
  not be accepted for the Attendance/Audit write itself, which is why that part uses a
  transaction rather than an event.
- Cross-database consistency between Biometric (MongoDB) and Attendance/Audit (PostgreSQL)
  is eventual, not ACID — this is acceptable because Biometric reads are precondition checks
  (match/no-match) that happen before the transactional write, and the system can tolerate
  the small window of inconsistency.

**Impact on the system:**
- Affected services: `Biometric` (read path, MongoDB), `Attendance` (PostgreSQL), `Audit`
  (PostgreSQL), `Configuration` (notification consumer).
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
| A future decision extracts Attendance or Audit to its own service without revisiting this ADR | Medium (natural temptation once the system grows) | High (silently breaks the atomicity guarantee this ADR relies on) | Add an explicit cross-reference from ADR-002's "Consequences" section to this ADR, so any extraction discussion is forced to reconsider it |
| Process crash between transaction commit and in-process event dispatch drops a notification | Low-Medium | Low (notifications are not safety-critical; a missed one does not corrupt attendance data) | Accept as a known limitation; consider Outbox pattern (already documented in `pattern-guide.md`) if notification reliability requirements increase |
| Long-running transaction holds locks if the write path grows to include more logic later | Low today | Medium if it happens | Keep the transaction scoped strictly to Attendance + Audit writes; any new logic added to this flow must be evaluated against this boundary before being included inside the transaction |
| Cross-database inconsistency between Biometric (MongoDB) and Attendance/Audit (PostgreSQL) during the read-before-write window | Low (Biometric data is relatively stable; reads are fast) | Medium (could lead to false positive/negative matches if Biometric data changes during the window) | Treat Biometric read as a point-in-time snapshot; accept eventual consistency for this specific cross-database interaction; document the consistency model in `09-microservices/communication-patterns.md` |

---

## References

- Use Cases this flow implements → SRS "Registro asistencia" (UC1) and "Escaneo facial" (UC2)
- Functional requirements → RF 3.1, RF 3.1.1, RF 3.1.2
- Table constraints → the system's entity-relationship model (`Class_Session`, `Attendance`)
- Mandatory audit logging → `00-governance/security-policy.md`
- Saga / Outbox pattern reference (for future extraction scenario) → `05-architecture/pattern-guide.md`
- Related to: ADR-002 (single-instance strategy this decision depends on for Attendance/Audit)
- Related to: ADR-003 (Biometric in MongoDB, excluded from PostgreSQL transaction)