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

---

## 2. Auth Service

| Event | Topic/Exchange | Producer | Consumers | Payload (key fields) |
|-------|---------------|----------|-----------|----------------------|
| AuthDomainUpdated | 02-auth-service.events | Auth | — | `{entityId, version, occurredAt}` |

---

## 3. Authorization Service

| Event | Topic/Exchange | Producer | Consumers | Payload (key fields) |
|-------|---------------|----------|-----------|----------------------|
| RoleCreated | authorization.role.events | Authorization | — | `{roleId, schoolId, roleName}` |
| PermissionAssigned | authorization.permission.events | Authorization | — | `{roleId, permissionId, assignmentDate}` |
| UserRoleGranted | authorization.user.events | Authorization | — | `{userId, roleId, assignmentDate}` |
| UserRoleRevoked | authorization.user.events | Authorization | — | `{userId, roleId}` |

---

## 4. Academic Service

| Event | Topic/Exchange | Producer | Consumers | Payload (key fields) |
|-------|---------------|----------|-----------|----------------------|
| CourseCreated | academic.course.events | Academic | — | `{courseId, schoolId, courseCode}` |
| EnrollmentCreated | academic.enrollment.events | Academic | — | `{enrollmentId, studentId, courseId, periodId}` |
| ScheduleCreated | academic.schedule.events | Academic | Attendance | `{scheduleId, courseId, teacherId, classroomId}` |
| StudentProfileCreated | academic.student.events | Academic | — | `{studentId, personId, schoolId}` |
| TeacherProfileCreated | academic.teacher.events | Academic | — | `{teacherId, personId, schoolId}` |

---

## 5. Attendance Service

| Event | Topic/Exchange | Producer | Consumers | Payload (key fields) |
|-------|---------------|----------|-----------|----------------------|
| ClassSessionOpened | attendance.session.events | Attendance | IoT | `{sessionId, scheduleId, date, openedBy}` |
| ClassSessionClosed | attendance.session.events | Attendance | — | `{sessionId, scheduleId, date, closedBy}` |
| AttendanceRecorded | attendance.attendance.events | Attendance | Audit | `{attendanceId, studentId, sessionId, status}` |
| AttendanceCaptured | attendance.attendance.events | Attendance | Biometric | `{studentId, sessionId, deviceId, captureType, timestamp}` |
| JustificationSubmitted | attendance.justification.events | Attendance | — | `{justificationId, attendanceId, approval}` |

---

## 6. Biometric Service

| Event | Topic/Exchange | Producer | Consumers | Payload (key fields) |
|-------|---------------|----------|-----------|----------------------|
| FacialMatchSucceeded | biometric.match.events | Biometric | — | `{subjectId, schoolId, confidence}` |
| FacialMatchFailed | biometric.match.events | Biometric | — | `{subjectId, schoolId, reason}` |
| FingerprintMatchSucceeded | biometric.fingerprint.events | Biometric | — | `{subjectId, schoolId, finger}` |
| BiometricVerified | biometric.verification.events | Biometric | Attendance | `{subjectId, schoolId, sessionId, matchType, confidence}` |
| FacialUpdateRequested | biometric.review.events | Biometric | — | `{caseId, subjectId, reason}` |
| FacialUpdateReviewed | biometric.review.events | Biometric | — | `{caseId, status, reviewedBy}` |

---

## 7. IoT Service

| Event | Topic/Exchange | Producer | Consumers | Payload (key fields) |
|-------|---------------|----------|-----------|----------------------|
| DeviceRegistered | iot.device.events | IoT | — | `{deviceId, schoolId, deviceName}` |
| DeviceAssigned | iot.assignment.events | IoT | — | `{deviceId, classroomId, assignedFrom}` |
| DeviceStatusChanged | iot.device.events | IoT | Audit | `{deviceId, status, lastConnection}` |
| DeviceOnline | iot.device.events | IoT | Attendance | `{deviceId, schoolId, classroomId, timestamp}` |
| DeviceHeartbeat | iot.telemetry.events | IoT | — | `{deviceId, status, timestamp}` |

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
| SecurityConfigurationChanged | configuration.security.events | Configuration | — | `{configurationName, configurationValue}` |
| PasswordPolicyChanged | configuration.security.events | Configuration | Identity | `{policyId, minLength, expirationDays}` |
| ConfigurationUpdated | configuration.system.events | Configuration | IoT | `{configurationType, configurationName, configurationValue, schoolId}` |

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
