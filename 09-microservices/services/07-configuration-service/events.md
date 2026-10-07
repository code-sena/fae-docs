# Events — Configuration Service

> The Configuration service emits policy and setting updates that affect the rest of the platform.

---

## Events this service PUBLISHES

| Event | Topic/Exchange | When it is emitted | Payload (key fields) |
|-------|---------------|--------------------|---------------------|
| AcademicConfigurationChanged | configuration.academic.events | An academic setting is updated | `{schoolId, configurationName, configurationValue}` |
| SecurityConfigurationChanged | configuration.security.events | A security rule or policy value changes | `{configurationName, configurationValue}` |
| PasswordPolicyChanged | configuration.security.events | Password rules are updated | `{policyId, minLength, expirationDays}` |
| ConfigurationUpdated | configuration.system.events | Any configuration value is updated | `{configurationType, configurationName, configurationValue, schoolId}` |

### Event schema: PasswordPolicyChanged

```json
{
  "eventId": "uuid",
  "eventType": "PasswordPolicyChanged",
  "timestamp": "2026-09-05T12:00:00Z",
  "version": "1.0",
  "source": "configuration-service",
  "payload": {
    "policyId": 4,
    "minLength": 8,
    "maxLength": 20,
    "expirationDays": 90
  }
}
```

---

## Events this service CONSUMES

| Event | Published by | Topic/Exchange | What this service does when it receives it |
|-------|-------------|----------------|-------------------------------------------|
| SchoolCreated | identity-service | identity.school.events | Creates or initializes default school-level configuration values |
| SystemMaintenance | platform-operations | operations.system.events | Loads or pauses configuration propagation for maintenance windows |

---

## Delivery guarantees

| Guarantee | Value | Implication |
|-----------|-------|-------------|
| At-least-once | Yes | Consumers should treat configuration updates as idempotent refreshes |
| At-most-once | No | Some updates may be replayed during deployment windows |
| Exactly-once | No | Duplicate updates are resolved by version or latest value semantics |

---

## Error handling

- **Dead Letter Queue:** `configuration.dlq`
- **Retries:** 5 retries with exponential backoff
- **Alerts:** Notify platform owners if configuration changes fail to propagate to dependent services
