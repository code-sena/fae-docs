# Domain Events

> A domain event is a fact that occurred in the business.
> They are the backbone of asynchronous communication between bounded contexts.
> The name is ALWAYS in past tense and in the ubiquitous language of the domain.

---

## What is a domain event?

| Event | Trigger | Side effect |
|---|---|---|
| `UserRegistered` | A record is inserted into `app_user`. | The user can now authenticate. An audit record is created. |
| `UserActivated` | `status` in `app_user` is set to active. | The user can access system functions. |
| `UserDeactivated` | `status` in `app_user` is set to inactive. | The user cannot authenticate. Active sessions are invalidated. |
| `SessionStarted` | A record is inserted into `user_session`. | The user can interact with the system for a limited period. |
| `PasswordChanged` | `password_hash` in `app_user` is updated. | Previous credentials are invalidated. An audit record is created. |
| `RoleAssigned` | A record is inserted into `user_role`. | The user gains the permissions associated with the new role. |
| `SchoolRegistered` | A record is inserted into `school`. | The school becomes available for academic data. |
| `ProgramCreated` | A record is inserted into `program`. | Cohorts and courses can be associated with the program. |
| `CohortRegistered` | A record is inserted into `cohort`. | Students can be enrolled and attendance can be tracked. |
| `EnrollmentCreated` | A record is inserted into `enrollment`. | The student appears in the cohort's attendance roster. |
| `EnvironmentRegistered` | A record is inserted into `environment`. | The environment can be assigned to schedule blocks. |
| `ScheduleBlockCreated` | A record is inserted into `schedule_block`. | Class sessions can be generated from this block. |
| `ClassSessionOpened` | `status` in `class_session` is set to Open. | Attendance can be recorded for this session. |
| `ClassSessionClosed` | `status` in `class_session` is set to Closed. | No more attendance can be recorded. Reports can be generated. |
| `FacialEmbeddingEnrolled` | A document is inserted into `facial_embeddings`. | The person can be identified by facial recognition. |
| `FacialEmbeddingUpdated` | A document in `facial_embeddings` is updated. | The person's facial template version increments. |
| `FingerprintEmbeddingEnrolled` | A document is inserted into `fingerprint_embeddings`. | The person can be identified by fingerprint. |
| `AttendanceRecorded` | A record is inserted into `attendance_record`. | It calculates whether it is "Late" by comparing `captured_at` with `schedule_block` times. The Dashboard is updated. |
| `LateArrivalDetected` | `attendance_status` is set to Late. | An alert may be raised for the instructor. |
| `AbsenceDetected` | `attendance_status` is set to Absent after session closes. | An alert may be raised. The student can submit a justification. |
| `JustificationRequested` | A record is inserted into `justification` (status = Pending). | A notification is sent to the reviewer. |
| `SupportingDocumentUploaded` | A record is inserted into `supporting_document`. | The justification file is available for review. |
| `JustificationApproved` | `review_status` in `justification` is updated to Approved. | The `attendance_status` is updated to "Justified". The student is notified. |
| `JustificationRejected` | `review_status` in `justification` is updated to Rejected. | The student is notified. The absence record remains. |
| `AlertRaised` | A record is inserted into `alert`. | The receiving user is notified. |
| `AlertResolved` | `resolved_at` in `alert` is set. | The alert is marked as resolved. |
| `BiometricUpdateRequested` | A record is inserted into `biometric_update_case` (status = Pending). | An administrator is notified to review. |
| `BiometricUpdateApproved` | `update_status` in `biometric_update_case` is updated to Approved. | The biometric embedding can be updated. |
| `BiometricUpdateRejected` | `update_status` in `biometric_update_case` is updated to Rejected. | The requestor is notified. No embedding change. |
| `ActionAudited` | A record is inserted into `audit_log`. | The action is permanently recorded for traceability. |
| `ErrorLogged` | A record is inserted into `error_log`. | The error is available for debugging and monitoring. |

---

## Resilience patterns for events

### At-least-once delivery + Idempotency

The message broker guarantees the event is delivered **at least once** but it may be
delivered more than once (in case of retries). Consumers must be **idempotent**.

```typescript
// Idempotent consumer — stores the processed eventId
async function processEvent(event: DomainEvent): Promise<void> {
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
