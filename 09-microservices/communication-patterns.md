# Communication Patterns

> How services communicate with each other and with external clients. The platform uses a hybrid model: synchronous APIs for real-time queries and command orchestration, and asynchronous events for eventual consistency.

---

## 1. Synchronous communication

Used when a client or service needs an immediate response.

| Aspect | Detail |
|--------|--------|
| **Protocol** | HTTP/REST over TLS |
| **Entry point** | API Gateway (port 8080) |
| **Consumer** | Web, Mobile, and external API clients |
| **Pattern** | Request → Response |
| **Use cases** | Queries, command orchestration, login, CRUD operations |

### When to use synchronous

- The caller requires an immediate answer (e.g., login, attendance query).
- The operation is a direct command that must succeed or fail in the request scope.
- The interaction is between the API Gateway and a domain service.

### Synchronous flow

```
Client ──▶ API Gateway ──▶ Domain Service
                              │
                              ▼
                           Database
                              │
Client ◀── API Gateway ◀──────┘
```

---

## 2. Asynchronous communication

Used when services need to propagate state changes without blocking.

| Aspect | Detail |
|--------|--------|
| **Transport** | Kafka or RabbitMQ |
| **Pattern** | Publish → Subscribe (event-driven) |
| **Consistency** | Eventual |
| **Use cases** | Cross-context state propagation, audit logging, notifications |

### When to use asynchronous

- The producer does not need to wait for the consumer to process the event.
- Multiple services need to react to the same state change.
- The operation can tolerate a small delay (seconds to low minutes).
- The producer should not fail if a consumer is temporarily unavailable.

### Asynchronous flow

```
Producer Service ──▶ Event Bus (Kafka/RabbitMQ) ──▶ Consumer Service A
                                                  ──▶ Consumer Service B
                                                  ──▶ Audit Service
```

---

## 3. External ID references

Services never use direct database joins across boundaries. Instead, they reference entities by public IDs.

| Pattern | Description | Example |
|---------|-------------|---------|
| Public ID reference | Store the UUID of an entity owned by another context | `Attendance.student_id` holds the Identity `user_id` UUID |
| Local projection | Cache a materialized copy of cross-context data for read performance | Academic stores person names locally for display |
| API call on demand | Fetch cross-context data in real time when freshness is required | Attendance calls Academic API to verify enrollment status |

### Rules

1. Cross-context references are always public UUIDs, never auto-increment keys.
2. Referencing services must handle the case where the referenced entity does not yet exist (eventual consistency).
3. No service may issue SQL joins against another service's database.

---

## 4. Communication matrix

| Source | Target | Channel | Pattern | Purpose |
|--------|--------|---------|---------|---------|
| API Gateway | Identity Service | HTTP/REST | Sync | Authentication, user CRUD |
| API Gateway | Authorization Service | HTTP/REST | Sync | Role/permission checks |
| API Gateway | Academic Service | HTTP/REST | Sync | Academic entity CRUD |
| API Gateway | Attendance Service | HTTP/REST | Sync | Attendance queries and commands |
| API Gateway | Biometric Service | HTTP/REST | Sync | Biometric enrollment and lookup |
| API Gateway | IoT Service | HTTP/REST | Sync | Device management |
| API Gateway | Audit Service | HTTP/REST | Sync | Audit log queries |
| API Gateway | Configuration Service | HTTP/REST | Sync | Configuration reads |
| Identity Service | Authorization Service | Event | Async | User and school lifecycle |
| Identity Service | Academic Service | Event | Async | Person creation |
| Identity Service | Configuration Service | Event | Async | School creation |
| Academic Service | Attendance Service | Event | Async | Schedule creation |
| Attendance Service | Audit Service | Event | Async | Attendance recording |
| Biometric Service | Attendance Service | Event | Async | Biometric verification |
| IoT Service | Attendance Service | Event | Async | Device online status |
| Configuration Service | Academic Service | Event | Async | Configuration changes |
| Configuration Service | Identity Service | Event | Async | Policy changes |
| Audit Service (consumes) | Identity, Attendance, IoT | Event | Async | Traceability |

---

## 5. Error handling patterns

### Synchronous errors

| HTTP Code | Meaning | Action |
|-----------|---------|--------|
| 400 | Bad request / validation error | Return error details to caller |
| 401 | Unauthorized | Caller must re-authenticate |
| 403 | Forbidden | Caller lacks required permissions |
| 404 | Not found | Resource does not exist in the owning context |
| 409 | Conflict | Business rule violation (e.g., duplicate enrollment) |
| 422 | Unprocessable | Valid JSON but semantically incorrect |
| 429 | Rate limited | Caller must back off |
| 5xx | Server error | Retry with exponential backoff, alert operations |

### Asynchronous errors

| Mechanism | Description |
|-----------|-------------|
| Dead Letter Queue (DLQ) | Failed events are routed to a per-service DLQ for manual inspection |
| Retry with backoff | 4–6 retries with exponential backoff before DLQ |
| Idempotent handlers | All consumers must be idempotent to handle at-least-once delivery |
| Alerting | DLQ depth or retry count exceeding threshold triggers an alert |

### Per-service DLQs

| Service | DLQ Topic |
|---------|-----------|
| Identity | `identity.dlq` |
| Auth | `auth.dlq` |
| Authorization | `authorization.dlq` |
| Academic | `academic.dlq` |
| Attendance | `attendance.dlq` |
| Biometric | `biometric.dlq` |
| IoT | `iot.dlq` |
| Audit | `audit.dlq` |
| Configuration | `configuration.dlq` |

---

## 6. Interaction style summary

| Style | When | Latency | Consistency | Complexity |
|-------|------|---------|-------------|------------|
| Sync (HTTP/REST) | Real-time queries, commands | Low (ms) | Strong (request scope) | Low |
| Async (Events) | Cross-context propagation | Medium (ms–s) | Eventual | Medium |
| Hybrid | Orchestrate sync commands, then publish events | Mixed | Mixed | High |

---

## Correlations

- Service catalog and communication matrix → `09-microservices/service-catalog.md`
- Event catalog → `09-microservices/event-catalog.md`
- OpenAPI contracts → `07-api/contracts/openapi/`
- Domain events → `02-domain/domain-events.md`
