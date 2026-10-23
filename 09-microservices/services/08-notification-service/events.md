# Events — Notification Service

> **Publishes nothing; consumes two topics** (two consumers, two consumer groups). Verified
> against `FULL/back-end/08-ms-notification` on 2026-10-23.

---

## Events this service PUBLISHES

**None.** No Kafka producer exists in the code (`package.json` declares `kafkajs` for the
consumer only; no `producer()` call anywhere in `src/`).

## Events this service CONSUMES

**Two topics**, each with its own consumer (kafkajs, `fromBeginning: false`).

### 1. `attendance.justification.events` — group `notification-service`

- **Producer:** `05-ms-attendance` — `ReviewJustificationUseCaseImpl.TOPIC`, published
  **after commit** when a justification is approved or rejected.
- **Handler:** `src/infrastructure/messaging/justification.consumer.ts`. On a
  `JustificationReviewed` event (any other `eventType` is silently ignored) it resolves the alert type by code
  (`JUSTIFICATION_APPROVED` / `JUSTIFICATION_REJECTED`) and raises the DASHBOARD alert the
  apprentice sees in Novedades (ERF 6.3).
- **Idempotent** on the natural key (actor + alert type + `raisedAt`), so at-least-once
  redelivery does not duplicate the alert.
- **Retry:** in-place exponential backoff inside `eachMessage`, capped at 30 s; the offset is
  never committed until the handler succeeds. Non-retryable events (bad status, zero actor id,
  empty `reviewedAt`, unparseable JSON) are logged and skipped, not retried.
- **Opt-in:** this consumer only starts when `KAFKA_BROKERS` is set. Without it the service
  boots normally with no consumer.
- **No DLQ, no outbox, no compensation** — platform-wide gaps recorded as `AT-006`.
- **Alert-type seeds:** the handler looks up `JUSTIFICATION_APPROVED` / `JUSTIFICATION_REJECTED`
  through `alertTypes.findByCode`; an absent row would throw a retryable `Error`. Both codes are
  seeded by the incremental changeset
  `database/08-ms-notification-db/02-dml/003-seed-justification-result-alert-types.yaml`
  (INFO, DASHBOARD), so a freshly migrated database does not hit that path.
- The payload also carries `version` and `attendanceRecordId`; `academicActorId` may be `null`,
  which lands in the "zero actor id" skip path.

### 2. `scheduling.class-session.closed` — group `notification-service-class-session`

- **Producer:** `04-ms-scheduling` — `CloseClassSessionUseCaseImpl.CLOSED_TOPIC`, published when a
  class session is closed.
- **Handler:** `src/infrastructure/messaging/classSession.consumer.ts` →
  `handleClassSessionClosed` (`application/attendanceAlerts/absenceReport.ts`): records the
  absences of the class and e-mails the instructor.
- **Same retry contract** as the justification consumer (in-place exponential backoff capped at
  30 s; invalid events skipped). Its start is also retried while the broker or topic does not
  exist yet: nothing publishes the topic until the first class closes.
- **Gating:** starts only when `KAFKA_BROKERS` **and** the service-account credentials
  `NOTIFICATION_SERVICE_USERNAME` / `NOTIFICATION_SERVICE_PASSWORD` are set (it needs them to
  read the other services through `HttpPeers`). The same credentials enable the daily at-risk
  scheduler (`RISK_ALERT_HOUR`, `DAILY_SUMMARY_HOUR`, `ATTENDANCE_ALERTS_ENABLED`).

> `SERVICE.md` §5 ("Consumo de Eventos Kafka") still shows a `kafka.Consume(...)` snippet for
> `attendance-events` / `scheduling-events` / `configuration-events` / `identity-events` —
> **that code does not exist**; it documents a target state. Only the two topics above are
> real. The full platform inventory is in `../communication-patterns.md` §2.

---

## How alerts are actually created and delivered

| Path | Direction | Detail |
|------|-----------|--------|
| `POST /api/v1/alerts` (and alert-type CRUD) | **In** — REST through Kong | API clients create/resolve alert rows; dashboards read them |
| Kafka `attendance.justification.events` | **In** — async | Justification review → dashboard alert (see above); only when `KAFKA_BROKERS` is set |
| Kafka `scheduling.class-session.closed` | **In** — async | Class closed → absences recorded + instructor e-mail; needs `KAFKA_BROKERS` and the service-account credentials |
| `POST /api/v1/emails` | **In** — internal REST | Consumed by Identity for password-reset/verification mails; optional `X-Internal-Token` (`EMAIL_INTERNAL_TOKEN`) |
| `WEBHOOK_URL` (optional) | **Out** — external | `POST` per new alert, **only** for `EMAIL`/`PUSH` alert types, fire-and-forget and detached from the request; HMAC-SHA256 signed when `WEBHOOK_SECRET` is set. Skipped silently when `WEBHOOK_URL` is unset |
| SMTP delivery | **Out** — external | nodemailer; when `SMTP_HOST` is empty the email endpoint answers `202 {"status":"skipped","reason":"smtp_not_configured"}` |

> Push (Firebase) and SMS (Twilio) appear in `SERVICE.md` as planned channels — **no such
> imports exist in the code**; email is the only implemented channel.

---

## Delivery guarantees

- **Ingest (Kafka):** at-least-once. The justification handler is idempotent on
  (actor, alert type, `raisedAt`).
- **Egress (webhook, SMTP):** effectively at-most-once — failures are logged, there is no retry
  queue; the SMTP send itself returns accepted-and-skipped when not configured.

---

## Correlations

- Target catalog (what alerts *would* be event-driven) → `../event-catalog.md`
- Reality of async communication → `../communication-patterns.md` §2
- Relevant ADR → `05-architecture/decisions/records/ADR-006-message-broker.md` (target state)
- Rewrite that introduced this consumer → `05-architecture/decisions/records/ADR-021-notification-service-typescript.md`
