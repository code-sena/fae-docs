# Event Catalog

> Canonical list of business and operational events published across all services. Each event is the contract for asynchronous integration between bounded contexts.

---

## 1. Identity Service

| Event | Topic/Exchange | Producer | Consumers | Payload (key fields) |
|-------|---------------|----------|-----------|----------------------|
| SchoolCreated | identity.school.events | Identity | Authorization, Academic, Configuration | `{schoolId, nit, name}` |
| PersonCreated | identity.person.events | Identity | Academic | `{personId, schoolId, email}` |
| UserCreated | identity.user.events | Identity | Authorization, Biometric, Audit | `{userId, personId, username}` |
| UserSessionStarted | identity.session.events | Identity | — | `{sessionId, userId, sourceIp}` |
| UserSessionEnded | identity.session.events | Identity | — | `{sessionId, userId, sessionStatus}` |
| UserAuthenticated | identity.auth.events | Identity | Audit | `{userId, sessionId, success, sourceIp}` |

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
| ClassSessionClosed | scheduling.session.events | Scheduling | Audit | `{classSessionId, scheduleBlockId, sessionDate, closedBy}` |

---

## 5. Attendance Service

| Event | Topic/Exchange | Producer | Consumers | Payload (key fields) |
|-------|---------------|----------|-----------|----------------------|
| AttendanceRecorded | attendance.attendance.events | Attendance | Audit, Notification | `{attendanceId, studentId, sessionId, status, captureType}` |
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

## 8. Audit Service

| Event | Topic/Exchange | Producer | Consumers | Payload (key fields) |
|-------|---------------|----------|-----------|----------------------|
| AuditEventRecorded | audit.events | Audit | — | `{auditEventId, actorId, action, aggregateType}` |
| ErrorLogged | audit.errors | Audit | — | `{errorId, errorType, occurredAt}` |

---

## 9. Configuration Service

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
