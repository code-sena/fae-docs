# Events — Authorization Service

> The Authorization service emits permission and role changes that affect access evaluation across the platform.

---

## Events this service PUBLISHES

| Event | Topic/Exchange | When it is emitted | Payload (key fields) |
|-------|---------------|--------------------|---------------------|
| RoleCreated | authorization.role.events | A new role is created | `{roleId, schoolId, roleName}` |
| PermissionAssigned | authorization.permission.events | Permission is granted to a role | `{roleId, permissionId, assignmentDate}` |
| UserRoleGranted | authorization.user.events | A user is assigned to a role | `{userId, roleId, assignmentDate}` |
| UserRoleRevoked | authorization.user.events | A user-role mapping is removed | `{userId, roleId}` |

### Event schema: UserRoleGranted

```json
{
  "eventId": "uuid",
  "eventType": "UserRoleGranted",
  "timestamp": "2026-09-05T12:00:00Z",
  "version": "1.0",
  "source": "authorization-service",
  "payload": {
    "userId": "uuid",
    "roleId": 3,
    "schoolId": "uuid",
    "assignmentDate": "2026-09-05T12:00:00Z"
  }
}
```

---

## Events this service CONSUMES

| Event | Published by | Topic/Exchange | What this service does when it receives it |
|-------|-------------|----------------|-------------------------------------------|
| UserCreated | identity-service | identity.user.events | Validates whether a user may receive default role assignments |
| SchoolCreated | identity-service | identity.school.events | Initializes default role set or school-scoped policy templates |

---

## Delivery guarantees

| Guarantee | Value | Implication |
|-----------|-------|-------------|
| At-least-once | Yes | Consumers must be idempotent |
| At-most-once | No | Duplicate updates can be handled safely |
| Exactly-once | No | Role changes are resolved by natural keys |

---

## Error handling

- **Dead Letter Queue:** `authorization.dlq`
- **Retries:** 4 retries with exponential backoff
- **Alerts:** Trigger escalation when role assignment fails for a critical user or school scope
