# Events — Academic Service

> The Academic service emits changes to school teaching structures, enrollment, and scheduling.

---

## Events this service PUBLISHES

| Event | Topic/Exchange | When it is emitted | Payload (key fields) |
|-------|---------------|--------------------|---------------------|
| CourseCreated | academic.course.events | A course is created | `{courseId, schoolId, courseCode}` |
| EnrollmentCreated | academic.enrollment.events | A student enrolls in a course | `{enrollmentId, studentId, courseId, periodId}` |
| ScheduleCreated | academic.schedule.events | A schedule record is created | `{scheduleId, courseId, teacherId, classroomId}` |
| StudentProfileCreated | academic.student.events | A student academic profile is created | `{studentId, personId, schoolId}` |
| TeacherProfileCreated | academic.teacher.events | A teacher academic profile is created | `{teacherId, personId, schoolId}` |

### Event schema: EnrollmentCreated

```json
{
  "eventId": "uuid",
  "eventType": "EnrollmentCreated",
  "timestamp": "2026-09-05T12:00:00Z",
  "version": "1.0",
  "source": "academic-service",
  "payload": {
    "enrollmentId": 123,
    "studentId": "uuid",
    "courseId": 45,
    "periodId": 7,
    "schoolId": "uuid"
  }
}
```

---

## Events this service CONSUMES

| Event | Published by | Topic/Exchange | What this service does when it receives it |
|-------|-------------|----------------|-------------------------------------------|
| PersonCreated | identity-service | identity.person.events | Publishes a derived academic persona if the person is assigned to an academic profile |
| SchoolCreated | identity-service | identity.school.events | Initializes school-level academic catalogs and default periods |

---

## Delivery guarantees

| Guarantee | Value | Implication |
|-----------|-------|-------------|
| At-least-once | Yes | Consumers must be idempotent |
| At-most-once | No | Duplicate enrollment messages may need de-duping |
| Exactly-once | No | Business keys are used to avoid repeated inserts |

---

## Error handling

- **Dead Letter Queue:** `academic.dlq`
- **Retries:** 5 retries with exponential backoff
- **Alerts:** Trigger alert when `EnrollmentCreated` or `ScheduleCreated` events are delayed or requeued more than 3 times
