# Events — Notification Service

> **No events at all — neither published nor consumed.** Verified against
> `FULL/back-end/08-ms-notification` on 2026-10-17.

---

## Events this service PUBLISHES

**None.** No Kafka client exists in the Go code (`go.mod` and all `.go` files contain no
`segmentio/kafka-go`, `sarama` or equivalent import).

## Events this service CONSUMES

**None.** `SERVICE.md` §5 ("Consumo de Eventos Kafka") shows a `kafka.Consume(...)` snippet for
`attendance-events` / `scheduling-events` / `configuration-events` / `identity-events` — **that code
does not exist**; it documents a target state. The real inventory (all services, publish-only,
zero consumers) is in `../communication-patterns.md` §2.

---

## How alerts are actually created and delivered

| Path | Direction | Detail |
|------|-----------|--------|
| `POST /api/v1/alerts` (and alert-type CRUD) | **In** — REST through Kong | API clients create/resolve alert rows; dashboards read them |
| `POST /api/v1/emails` | **In** — internal REST | Consumed by Identity for password-reset/verification mails; optional `X-Internal-Token` (`EMAIL_INTERNAL_TOKEN`) |
| SMTP delivery | **Out** — external | `net/smtp` client; when `SMTP_HOST` is empty the endpoint answers `202 {"status":"skipped","reason":"smtp_not_configured"}` |

> Push (Firebase) and SMS (Twilio) appear in `SERVICE.md` as planned channels — **no such imports
> exist in the code**; email is the only implemented channel.

---

## Delivery guarantees

Not applicable — there is no broker traffic. The only at-most-once semantics in this service is the
SMTP send itself (mail not configured → accepted-and-skipped, no retry queue).

---

## Correlations

- Target catalog (what alerts *would* be event-driven) → `../event-catalog.md`
- Reality of async communication → `../communication-patterns.md` §2
- Relevant ADR → `05-architecture/decisions/records/ADR-006-message-broker.md` (target state)
