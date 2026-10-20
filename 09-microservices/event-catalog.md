# Event Catalog

> **Current state (verified 2026-10-17) — read this first.** The tables below are the **target
> design**; they were never built as written. What actually runs today:
>
> - **Flat topic names** `{service}-events` (e.g. `identity-events`, `attendance-events`) for most
>   publishers, not the `{context}.{entity}.events` pattern in §"Event naming convention". One
>   exception: `05-ms-attendance` publishes justification **reviews** on the canonical
>   `attendance.justification.events` topic (`ReviewJustificationUseCaseImpl.TOPIC`), while
>   justification **creation** still uses the flat `justification-events`.
> - **Payloads are minimal JSON** (ids + status), not the rich payloads shown below.
> - **There is exactly one consumer in the platform:** `08-ms-notification` subscribes to
>   `attendance.justification.events` (kafkajs, consumer group `notification-service`) and raises
>   the `JUSTIFICATION_APPROVED` / `JUSTIFICATION_REJECTED` dashboard alerts. No other service
>   subscribes to any topic. There is still **no transactional outbox, no DLQ and no compensation**
>   (ADR-004 is only partially implemented — recorded as tech debt `AT-006`).
> - The `08-ms-notification` consumer only runs when `KAFKA_BROKERS` is set; with the variable
>   absent the service starts without a consumer.
>
> | Service | Topic actually published | Published on |
> |---|---|---|
> | 01 identity | `identity-events` | user locked, password changed/reset |
> | 01 identity | `password-policy-events` | password policy created |
> | 02 authorization | `role-events`, `permission-events` | role/permission created |
> | 03 academic | `academic-events` | request-level (method/url/status) hook |
> | 04 scheduling | `environment-events`, `schedule-block-events`, `class-session-events` | entity created |
> | 05 attendance | `attendance-events`, `justification-events`, `justification-type-events`, `document-events`, `attendance.justification.events` | record created (flat) / justification reviewed (canonical) |
> | 07 configuration | `configuration-events` | request-level (method/url/status) hook |
> | 06 biometric · 09 quality | — | **publish nothing** |
> | 08 notification | — | **publishes nothing** (consumes `attendance.justification.events`) |
>
> See `09-microservices/communication-patterns.md` §2 for the corrected inventory.

---

> Canonical list of business and operational events published across all services. Each event is the contract for asynchronous integration between bounded contexts.

---

## 1. Identity Service

| Event | Topic/Exchange | Producer | Consumers | Payload (key fields) |
|-------|---------------|----------|-----------|----------------------|
| SchoolCreated | identity.school.events | Identity | Authorization, Academic, Configuration | `{schoolId, nit, name}` |
| PersonCreated | identity.person.events | Identity | Academic | `{personId, schoolId, email}` |
| UserCreated | identity.user.events | Identity | Authorization, Biometric | `{userId, personId, username}` |
| UserSessionStarted | identity.session.events | Identity | — | `{sessionId, userId, sourceIp}` |
| UserSessionEnded | identity.session.events | Identity | — | `{sessionId, userId, sessionStatus}` |
| UserAuthenticated | identity.auth.events | Identity | — | `{userId, sessionId, success, sourceIp}` |

---

## 2. Authorization Service

| Event | Topic/Exchange | Producer | Consumers | Payload (key fields) |
|-------|---------------|----------|-----------|----------------------|
| RoleCreated | authorization.role.events | Authorization | — | `{roleId, schoolId, roleName}` |
| PermissionAssigned | authorization.permission.events | Authorization | — | `{roleId, permissionId, assignmentDate}` |
| UserRoleGranted | authorization.user.events | Authorization | — | `{userId, roleId, assignmentDate}` |
| UserRoleRevoked | authorization.user.events | Authorization | — | `{userId, roleId}` |

---

## 3. Academic Service

| Event | Topic/Exchange | Producer | Consumers | Payload (key fields) |
|-------|---------------|----------|-----------|----------------------|
| CourseCreated | academic.course.events | Academic | — | `{courseId, schoolId, courseCode}` |
| EnrollmentCreated | academic.enrollment.events | Academic | — | `{enrollmentId, studentId, courseId, periodId}` |
| StudentProfileCreated | academic.student.events | Academic | — | `{studentId, personId, schoolId}` |
| TeacherProfileCreated | academic.teacher.events | Academic | — | `{teacherId, personId, schoolId}` |

---

## 4. Scheduling Service

| Event | Topic/Exchange | Producer | Consumers | Payload (key fields) |
|-------|---------------|----------|-----------|----------------------|
| ScheduleBlockCreated | scheduling.block.events | Scheduling | Attendance | `{scheduleBlockId, cohortId, courseId, environmentId, dayOfWeek}` |
| ClassSessionOpened | scheduling.session.events | Scheduling | Attendance | `{classSessionId, scheduleBlockId, sessionDate, openedBy}` |
| ClassSessionClosed | scheduling.session.events | Scheduling | — | `{classSessionId, scheduleBlockId, sessionDate, closedBy}` |

---

## 5. Attendance Service

| Event | Topic/Exchange | Producer | Consumers | Payload (key fields) |
|-------|---------------|----------|-----------|----------------------|
| AttendanceRecorded | attendance.attendance.events | Attendance | Notification | `{attendanceId, studentId, sessionId, status, captureType}` |
| JustificationSubmitted | attendance.justification.events | Attendance | — | `{justificationId, attendanceId, reason}` |
| JustificationReviewed | attendance.justification.events | Attendance | Notification | `{justificationId, status, reviewedBy}` |

---

## 6. Biometric Service

| Event | Topic/Exchange | Producer | Consumers | Payload (key fields) |
|-------|---------------|----------|-----------|----------------------|
| FacialMatchSucceeded | biometric.match.events | Biometric | — | `{subjectId, schoolId, confidence}` |
| FacialMatchFailed | biometric.match.events | Biometric | — | `{subjectId, schoolId, reason}` |
| BiometricVerified | biometric.verification.events | Biometric | Attendance | `{subjectId, schoolId, sessionId, matchType, confidence}` |
| FacialUpdateRequested | biometric.review.events | Biometric | — | `{caseId, subjectId, reason}` |
| FacialUpdateReviewed | biometric.review.events | Biometric | — | `{caseId, status, reviewedBy}` |

---

## 7. Notification Service

| Event | Topic/Exchange | Producer | Consumers | Payload (key fields) |
|-------|---------------|----------|-----------|----------------------|
| AlertRaised | notification.alert.events | Notification | — | `{alertId, alertTypeCode, academicActorId, raisedAt}` |
| AlertResolved | notification.alert.events | Notification | — | `{alertId, alertTypeCode, resolvedAt}` |

---

## 8. Configuration Service

| Event | Topic/Exchange | Producer | Consumers | Payload (key fields) |
|-------|---------------|----------|-----------|----------------------|
| AcademicConfigurationChanged | configuration.academic.events | Configuration | Academic | `{schoolId, configurationName, configurationValue}` |
| SecurityConfigurationChanged | configuration.security.events | Configuration | Identity | `{configurationName, configurationValue}` |
| PasswordPolicyChanged | configuration.security.events | Configuration | Identity | `{policyId, minLength, expirationDays}` |

---

## Event naming convention

| Component | Pattern | Example |
|-----------|---------|---------|
| Topic name | `{context}.{entity}.{level}` | `identity.user.events` |
| Event type | PascalCase verb + noun | `UserCreated` |
| Source field | `{service-name}` | `identity-service` |
| Version | Semantic versioning | `1.0` |

---

## Delivery guarantees (platform-wide)

| Guarantee | Value | Implication |
|-----------|-------|-------------|
| At-least-once | Yes | All consumers must be idempotent |
| At-most-once | No | Events are not intentionally dropped |
| Exactly-once | No | Idempotent handlers resolve duplicates |

---

## Correlations

- Per-service event details → `09-microservices/services/NN-name/events.md`
- Domain events → `02-domain/domain-events.md`
- Communication patterns → `09-microservices/communication-patterns.md`
