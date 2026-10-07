# Events — Biometric Service

> The Biometric service emits recognition and template-update events that support attendance capture and identity verification workflows.

---

## Events this service PUBLISHES

| Event | Topic/Exchange | When it is emitted | Payload (key fields) |
|-------|---------------|--------------------|---------------------|
| FacialMatchSucceeded | biometric.match.events | A face template is matched successfully | `{subjectId, schoolId, confidence}` |
| FacialMatchFailed | biometric.match.events | A face template match fails | `{subjectId, schoolId, reason}` |
| FingerprintMatchSucceeded | biometric.fingerprint.events | A fingerprint template matches | `{subjectId, schoolId, finger}` |
| BiometricVerified | biometric.verification.events | A biometric identification is confirmed and ready for attendance correlation | `{subjectId, schoolId, sessionId, matchType, confidence}` |
| FacialUpdateRequested | biometric.review.events | A user requests a face-template revision | `{caseId, subjectId, reason}` |
| FacialUpdateReviewed | biometric.review.events | A review decision is made | `{caseId, status, reviewedBy}` |

### Event schema: FacialMatchSucceeded

```json
{
  "eventId": "uuid",
  "eventType": "FacialMatchSucceeded",
  "timestamp": "2026-09-05T12:00:00Z",
  "version": "1.0",
  "source": "biometric-service",
  "payload": {
    "subjectId": "uuid",
    "schoolId": "uuid",
    "confidence": 0.98,
    "modelVersion": "v2.4"
  }
}
```

---

## Events this service CONSUMES

| Event | Published by | Topic/Exchange | What this service does when it receives it |
|-------|-------------|----------------|-------------------------------------------|
| AttendanceCaptured | attendance-service | attendance.attendance.events | Uses the subject and session context to validate biometric recognition requests |
| UserCreated | identity-service | identity.user.events | Prepares subject metadata needed for biometric correlation |

---

## Delivery guarantees

| Guarantee | Value | Implication |
|-----------|-------|-------------|
| At-least-once | Yes | Recognition handlers must be idempotent |
| At-most-once | No | Duplicate requests may be retried safely |
| Exactly-once | No | Biometric match evaluation is based on natural keys and versioned templates |

---

## Error handling

- **Dead Letter Queue:** `biometric.dlq`
- **Retries:** 6 retries with exponential backoff
- **Alerts:** Trigger if recognition events are retried or template validation fails repeatedly
