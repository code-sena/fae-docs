# Events — Attendance Service

> The Attendance service emits real class and attendance events for operational monitoring and downstream analysis.

---

## Events this service PUBLISHES

| Event | Topic/Exchange | When it is emitted | Payload (key fields) |
|-------|---------------|--------------------|---------------------|
| ClassSessionOpened | attendance.session.events | A class session starts | `{sessionId, scheduleId, date, openedBy}` |
| ClassSessionClosed | attendance.session.events | A class session is closed | `{sessionId, scheduleId, date, closedBy}` |
| AttendanceRecorded | attendance.attendance.events | A student attendance event is saved | `{attendanceId, studentId, sessionId, status}` |
| AttendanceCaptured | attendance.attendance.events | A biometric or IoT capture triggers attendance registration | `{studentId, sessionId, deviceId, captureType, timestamp}` |
| JustificationSubmitted | attendance.justification.events | A justification is created or updated | `{justificationId, attendanceId, approval}` |

### Event schema: AttendanceRecorded

```json
{
  "eventId": "uuid",
  "eventType": "AttendanceRecorded",
  "timestamp": "2026-09-05T12:00:00Z",
  "version": "1.0",
  "source": "attendance-service",
  "payload": {
    "attendanceId": 567,
    "studentId": "uuid",
    "sessionId": 245,
    "status": "Present",
    "source": "IoT"
  }
}
```

---

## Events this service CONSUMES

| Event | Published by | Topic/Exchange | What this service does when it receives it |
|-------|-------------|----------------|-------------------------------------------|
| ScheduleCreated | academic-service | academic.schedule.events | Builds the session schedule or validates class-day dependencies |
| DeviceOnline | iot-service | iot.device.events | Allows the service to validate if a classroom device is active for capture |
| BiometricVerified | biometric-service | biometric.recognition.events | Correlates successful identity recognition with attendance records |

---

## Delivery guarantees

| Guarantee | Value | Implication |
|-----------|-------|-------------|
| At-least-once | Yes | Consumers must be idempotent |
| At-most-once | No | Duplicate attendance events must be de-duped |
| Exactly-once | No | Event processing relies on unique child keys |

---

## Error handling

- **Dead Letter Queue:** `attendance.dlq`
- **Retries:** 5 retries with exponential backoff
- **Alerts:** Trigger alert when event replay or attendance write failures exceed service thresholds
