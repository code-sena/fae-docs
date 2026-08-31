# Domain Events

> **What to fill in here:** A domain event is a fact that occurred in the business.
> They are the backbone of asynchronous communication between bounded contexts.
> The name is ALWAYS in past tense and in the ubiquitous language of the domain.

---

## What is a domain event?
| :--- | :--- | :--- |
| `EnrollmentCreated` | A record is inserted into `Enrollment`. | The system generates the student's schedule and enables facial registration. |
| `FacialEmbeddingGenerated` | `Facial_Embedding` is inserted/updated. | The classroom IoT device downloads the new vector for offline/local recognition. |
| `AttendanceRegistered` | A record is inserted into `Attendance`. | It calculates whether it is "Late" by comparing `time` with `Schedule.start_time`. The Dashboard is updated. |
| `JustificationSubmitted` | A record is inserted into `Justification` (status = Pending). | A notification is sent to the Instructor/Supervisor (potential `reviewed_by`). |
| `JustificationResolved` | `approval` in `Justification` is updated to Approved/Rejected. | The `Attendance` status is updated to "Justified" and the student is notified. |
| `DeviceDisconnected` | `last_connection` in `IoT_Device` exceeds the time threshold. | An alert (RF 4.5) is triggered for the Administrator/Supervisor role. |

---

## Resilience patterns for events

### At-least-once delivery + Idempotency

The message broker guarantees the event is delivered **at least once** but it may be
delivered more than once (in case of retries). Consumers must be **idempotent**.

```typescript
// Idempotent consumer — stores the processed eventId
async function processOrderCreatedEvent(event: OrderCreated): Promise<void> {
  // 1. Check if already processed
  if (await isEventAlreadyProcessed(event.eventId)) {
    logger.info(`Event ${event.eventId} already processed, ignoring`);
    return;
  }

  // 2. Process the event
  await updateModel(event.payload);

  // 3. Mark as processed (in the same transaction)
  await markEventProcessed(event.eventId);
}
```

### Dead Letter Queue (DLQ)

When an event fails after N retries, it goes to the DLQ.

| Configuration | Recommended value |
|--------------|------------------|
| Retries before DLQ | 3-5 |
| Backoff | Exponential (1s → 2s → 4s → 8s) |
| DLQ retention | 7 days |
| Alert | When DLQ has > 0 messages |

> See DLQ runbook in `09-microservices/services/XX-service/runbook.md`
