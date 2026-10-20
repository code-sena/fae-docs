# Events — Scheduling Service

> Publish-only on three flat topics, one per created entity. Verified against
> `FULL/back-end/04-ms-scheduling/src/main` on 2026-10-17.

---

## Events this service PUBLISHES (real)

| Event | Topic | When it is emitted | Payload (key fields) | Source |
|-------|-------|--------------------|----------------------|--------|
| EnvironmentCreated | `environment-events` | A room/lab is created | `{"environmentId":<n>,"code":"<code>"}` | `CreateEnvironmentUseCaseImpl` |
| ScheduleBlockCreated | `schedule-block-events` | A weekly block is created | `{"scheduleBlockId":<n>,"cohortId":<n>}` | `CreateScheduleBlockUseCaseImpl` |
| ClassSessionCreated | `class-session-events` | A session is materialized | `{"classSessionId":<n>,"scheduleBlockId":<n>}` | `CreateClassSessionUseCaseImpl` |

### Event shape: ClassSessionCreated (as emitted)

```json
{"classSessionId": 42, "scheduleBlockId": 7}
```

> **Payloads are minimal (ids only)** — no `eventId`/`eventType`/`version` envelope and no status
> fields. The rich envelope shown in `../event-catalog.md` is the **target** design, not what the
> code sends today.

---

## Events this service CONSUMES

**None.** There is no `@KafkaListener` (or any consumer) in `04-ms-scheduling` — verified
2026-10-17. Attendance reaches this service through its REST API (via Kong), not via events.

---

## Delivery guarantees

| Guarantee | Value | Implication |
|-----------|-------|-------------|
| Best-effort publish | Yes — publish failures are swallowed | No retries, no transactional outbox |
| At-least-once | No | Consumers cannot assume redelivery |
| Exactly-once | No | — |

### Error handling

- **No DLQ, no retries, no outbox** — tracked as tech debt `AT-006` (ADR-004 only partially
  implemented).
- **No consumer exists anywhere in the platform** (see `../communication-patterns.md` §2): today
  these topics are an audit trail, not an integration mechanism.

---

## Correlations

- Target catalog → `../event-catalog.md`
- Sync vs async reality → `../communication-patterns.md`
- Relevant ADR → `05-architecture/decisions/records/ADR-006-message-broker.md`
