# Events — Audit Service

> The Audit service records operational and business events for traceability, forensic review, and compliance.

---

## Events this service PUBLISHES

| Event | Topic/Exchange | When it is emitted | Payload (key fields) |
|-------|---------------|--------------------|---------------------|
| AuditEventRecorded | audit.events | A business or operational event is recorded | `{auditEventId, actorId, action, aggregateType}` |
| ErrorLogged | audit.errors | A system or application error is stored | `{errorId, errorType, occurredAt}` |

### Event schema: AuditEventRecorded

```json
{
  "eventId": "uuid",
  "eventType": "AuditEventRecorded",
  "timestamp": "2026-09-05T12:00:00Z",
  "version": "1.0",
  "source": "audit-service",
  "payload": {
    "auditEventId": 205,
    "actorId": "uuid",
    "action": "UserRoleGranted",
    "aggregateType": "Authorization",
    "aggregateId": "role:9"
  }
}
```

---

## Events this service CONSUMES

| Event | Published by | Topic/Exchange | What this service does when it receives it |
|-------|-------------|----------------|-------------------------------------------|
| UserCreated | identity-service | identity.user.events | Stores the identity lifecycle event for auditing |
| AttendanceRecorded | attendance-service | attendance.attendance.events | Records the attendance transaction as an audit trail |
| ClassSessionClosed | scheduling-service | scheduling.session.events | Logs class session closure for traceability |

---

## Delivery guarantees

| Guarantee | Value | Implication |
|-----------|-------|-------------|
| At-least-once | Yes | Audit records are append-only and should be retried safely |
| At-most-once | No | Duplicate events may arrive during replay or retry |
| Exactly-once | No | The system uses event keys and idempotent handling on write |

---

## Error handling

- **Dead Letter Queue:** `audit.dlq`
- **Retries:** 6 retries with exponential backoff
- **Alerts:** Trigger when audit writes fail or critical events are not persisted within the required window
