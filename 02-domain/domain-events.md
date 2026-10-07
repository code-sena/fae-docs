# Domain Events

> A domain event is a fact that occurred in the business.
> They are the backbone of asynchronous communication between bounded contexts.
> The name is ALWAYS in past tense and in the ubiquitous language of the domain.
> Transport: Kafka (see `05-architecture/decisions/records/ADR-006`).

---

## Event Catalog by Context (8 bounded contexts)

| Context | Event | Business trigger | Side effect / Consumer |
|---|---|---|---|
| Identity | `UserRegistered` | Administrator completes learner/instructor registration | Identity creates credentials; Authorization assigns default role |
| Identity | `UserActivated` | Administrator re-enables a disabled account | User regains access to all contexts |
| Identity | `UserDeactivated` | Administrator disables account (e.g., leave, sanction) | Active sessions invalidated; attendance capture blocked |
| Identity | `SessionStarted` | Learner/instructor signs in successfully | Session available for API Gateway; last_access updated |
| Identity | `PasswordChanged` | User completes password change flow | Previous sessions remain but credentials rotated |
| Authorization | `RoleAssigned` | Administrator assigns Role to App_User for a School | Permissions are looked up on the next `GET /api/v1/auth/evaluate` (no JWT claim to refresh) |
| Authorization | `RoleRemoved` | Role revoked | Permissions recalculated |
| Academic | `SchoolRegistered` | New institution onboarded | Enables programs, cohorts, environments |
| Academic | `ProgramCreated` | Academic coordinator creates program | Cohorts/courses can reference it |
| Academic | `CohortRegistered` | Cohort linked to Program+Period+School | Enrollment and scheduling enabled |
| Academic | `EnrollmentCreated` | Learner enrolled in cohort | Appears in roster; attendance can be recorded |
| Scheduling | `EnvironmentRegistered` | Classroom/lab registered for a School | Available for schedule blocks |
| Scheduling | `ScheduleBlockCreated` | Coordinator defines recurring slot (day/time/instructor/environment) | Class sessions can be generated; conflict check RN-18 |
| Scheduling | `ClassSessionOpened` | Instructor opens session for today | Attendance capture allowed (RN-23) |
| Scheduling | `ClassSessionClosed` | Instructor/system closes session | No further capture; Absence detection runs; reports available |
| Biometric | `FacialEmbeddingEnrolled` | Biometric enrollment succeeds for Person | Person can use FACIAL capture |
| Biometric | `FacialEmbeddingUpdated` | Re-enrollment approved via Biometric_Update_Case | Template version++ ; old embedding deactivated |
| Biometric | `FingerprintEmbeddingEnrolled` | Fingerprint captured (1..10 per Person) | Alternative capture available |
| Biometric | `BiometricUpdateRequested` | Learner requests embedding update (Pending) | Administrator notified |
| Biometric | `BiometricUpdateApproved` | Administrator approves case | Embedding may be re-enrolled |
| Biometric | `BiometricUpdateRejected` | Administrator rejects case | No change; requestor notified |
| Attendance | `AttendanceRecorded` | Learner captured via FACIAL/MANUAL/IMPORT during Open session | Late calculation vs Schedule_Block; Notification may raise alert |
| Attendance | `LateArrivalDetected` | System marks arrival after tardy tolerance | Alert to instructor |
| Attendance | `AbsenceDetected` | Session closes without record for enrolled actor | Alert; learner may submit Justification |
| Attendance | `JustificationRequested` | Learner submits justification for an Absence | Review task for instructor/admin |
| Attendance | `SupportingDocumentUploaded` | File attached to justification | Available for review |
| Attendance | `JustificationApproved` | Reviewer approves justification | Attendance status → Justified; learner notified |
| Attendance | `JustificationRejected` | Reviewer rejects justification | Absence remains; learner notified |
| Notification | `AlertRaised` | Threshold/conflict detected | Recipient notified in-app |
| Notification | `AlertResolved` | Authorized user resolves alert | Alert closed |

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
