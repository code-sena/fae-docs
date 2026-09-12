# Events — Identity Service

> The Identity service emits lifecycle events for school, person, user, and session state.

---

## Events this service PUBLISHES

| Event | Topic/Exchange | When it is emitted | Payload (key fields) |
|-------|---------------|--------------------|---------------------|
| SchoolCreated | identity.school.events | A new school is registered | `{schoolId, nit, name}` |
| PersonCreated | identity.person.events | A person record is created | `{personId, schoolId, email}` |
| UserCreated | identity.user.events | A user account is created | `{userId, personId, username}` |
| UserSessionStarted | identity.session.events | A login session begins | `{sessionId, userId, sourceIp}` |
| UserSessionEnded | identity.session.events | A session is closed or expired | `{sessionId, userId, sessionStatus}` |

### Event schema: UserCreated

```json
{
  "eventId": "uuid",
  "eventType": "UserCreated",
  "timestamp": "2026-09-05T12:00:00Z",
  "version": "1.0",
  "source": "identity-service",
  "payload": {
    "userId": "uuid",
    "personId": "uuid",
    "schoolId": "uuid",
    "username": "string",
    "authenticationType": "Local|Windows|External"
  }
}
```

---

## Events this service CONSUMES

| Event | Published by | Topic/Exchange | What this service does when it receives it |
|-------|-------------|----------------|-------------------------------------------|
| SchoolConfigurationChanged | configuration-service | configuration.school.events | Updates the current school configuration metadata if needed |
| PasswordPolicyChanged | configuration-service | security.policy.events | Refreshes runtime validation rules for password and login policies |

---

## Delivery guarantees

| Guarantee | Value | Implication |
|-----------|-------|-------------|
| At-least-once | Yes | Consumers must be idempotent |
| At-most-once | No | Events are not intentionally dropped |
| Exactly-once | No | The service relies on idempotent handlers |

---

## Error handling

- **Dead Letter Queue:** `identity.dlq`
- **Retries:** 5 retries with exponential backoff
- **Alerts:** Notify the identity operations owner if `UserCreated` or `UserSessionStarted` is retried beyond the threshold
