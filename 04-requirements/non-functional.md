# Non-Functional Requirements (NFR)

> Status: 🟡 In definition | Last updated: 2026-06-16  
> Author: Jonattan Rizo | Team: FaceAttendEDU  
> Primary source: SRS — FaceAttend EDU

> NFRs define the **qualities of the system** — not what it does but how well it does it.  
> The golden rule: every NFR must have a metric.  
> “The system must be fast” is not an NFR.  
> “The P95 response time for attendance registration must be under 5 seconds under expected pilot load” is.

---

## Purpose of this document

This document defines the non-functional requirements for **FaceAttend EDU**. These requirements describe how the system must behave in terms of:

- Performance.
- Availability.
- Scalability.
- Security and data protection.
- Compatibility.
- Usability and accessibility.
- Observability.
- Maintainability.
- Portability.
- Disaster recovery.
- Data integrity and auditability.

These requirements are derived from the SRS non-functional requirements RNF 1 to RNF 6, as well as architectural and engineering needs required to operate a facial-recognition attendance system responsibly.

---

## SRS mapping

| Internal NFR | SRS source | Category | Description |
|---|---|---|---|
| NFR-001 | RNF 1 | Performance | Response time, page load time, processing capacity, and recovery behavior. |
| NFR-002 | RNF 2, RNF 6 | Availability | System availability, continuity, and tolerance to failures. |
| NFR-003 | RNF 5 | Scalability | Ability to support user growth and traffic peaks. |
| NFR-004 | RNF 4 | Security | Authentication, encryption, privacy, and data-protection requirements. |
| NFR-005 | RNF 3 | Compatibility | Compatibility with browsers, mobile operating systems, devices, and screens. |
| NFR-006 | RNF 2 | Usability / Accessibility | Easy learning, clarity, error tolerance, accessibility, and language support. |
| NFR-007 | Engineering requirement | Observability | Logs, metrics, alerts, traces, and auditability. |
| NFR-008 | Engineering requirement | Maintainability | Test coverage, code quality, documentation, and onboarding. |
| NFR-009 | RNF 6, Engineering requirement | Portability | Deployment portability and environment independence. |
| NFR-010 | RNF 1, Engineering requirement | Disaster recovery | Recovery time, recovery point, backups, and resilience. |
| NFR-011 | Business and security requirement | Data integrity and auditability | Protection of attendance records, justification evidence, and administrative changes. |

---

## How to write a measurable NFR?

| Bad | Good |
|---|---|
| “The system must be fast.” | “The P95 response time for attendance registration must be under 5 seconds under expected pilot load.” |
| “The system must be secure.” | “All private endpoints require authenticated users, sensitive data must be encrypted, and communication must use HTTPS.” |
| “The system must scale.” | “The system must support a potential 50% increase in users within less than one year without significant degradation.” |
| “The system must be available.” | “The pilot environment should target 99.9% monthly availability, with a maximum of 44 minutes downtime per month, subject to validation.” |
| “Facial recognition must work well.” | “Facial verification should respond in under 5 seconds under normal lighting and camera conditions, with an alternative flow when recognition fails.” |

---

## NFR-001: Performance

**SRS source:** RNF 1 — Performance.  
**Priority:** P1.  
**Verification method:** Load testing, functional timing tests, browser performance testing, and pilot monitoring.

### Performance targets

| Attribute | Target | Test condition | Notes |
|---|---:|---|---|
| P95 latency — critical endpoints | < 2000 ms | Under expected pilot load | Proposed target. SRS requires response under 5 seconds. |
| P99 latency — critical endpoints | < 5000 ms | Under expected pilot load | Aligns with SRS maximum response time. |
| P95 latency — non-critical endpoints | < 5000 ms | Normal load | Based on SRS response-time requirement. |
| Web page load time | < 5 seconds | Standard broadband connection | Direct SRS requirement. |
| Mobile app initial load | < 5 seconds | Supported mobile device | Proposed target aligned with SRS. |
| Facial verification response | < 5 seconds | Normal camera, lighting, and network conditions | Derived from SRS performance and use-case constraints. |
| Attendance registration throughput | ≥ 30 successful registrations per minute | Simulated entry peak for one cohort | Proposed based on SRS justification of 30+ users per instructor. |
| Report generation time | < 10 seconds | Standard report with filters | Proposed target; to be validated with real data volume. |
| Justification file upload | < 10 seconds | Files up to 5 MB | Proposed target; maximum file size must be defined. |
| Service startup time | < 60 seconds | Cold start | Proposed engineering target. |
| Recovery from interface failure | < 5 minutes | Failure simulation | Direct SRS requirement; no information loss allowed. |

### Defined critical endpoints

> These endpoints are proposed based on the expected functionality of FaceAttend EDU. They must be confirmed once the API design is defined.

| Endpoint | Why it is critical |
|---|---|
| `POST /auth/login` | Users cannot access the system if authentication fails. |
| `POST /attendance/entry` | Core business operation: attendance entry registration. |
| `POST /attendance/exit` | Core business operation: attendance exit registration. |
| `POST /attendance/facial-verification` | Facial recognition is the main attendance capture method. |
| `POST /attendance/alternative-registration` | Required when facial recognition fails. |
| `GET /attendance/history` | Users and instructors need to consult attendance records. |
| `POST /justifications` | Students must be able to justify absences or late arrivals. |
| `GET /reports` | Instructors, administrators, and supervisors need timely reports. |
| `GET /health` | Required to validate that the service is alive. |
| `GET /health/ready` | Required to validate that the service can process traffic. |

## NFR-002: Availability

**SRS source:** RNF 2 — Availability, RNF 6 — Portability.  
**Priority:** P1.  
**Verification method:** Monitoring, health checks, availability logs, and incident records.

### Health checks

| Endpoint | Type | Expected behavior |
|---|---|---|
| `GET /health` | Liveness | Returns HTTP 200 if the process is alive. |
| `GET /health/ready` | Readiness | Returns HTTP 200 only if the system can process traffic and dependencies are available. |

### Dependency availability considerations

FaceAttend EDU may depend on:

- Internet connectivity.
- Functional cameras or mobile devices.
- External facial recognition services and other biometric data, such as fingerprints, if used.
- Database availability.
- Notification services.

If an external dependency fails, the system must show a clear error message and, where possible.

---

## NFR-003: Scalability

**SRS source:** RNF 5 — Scalability.  
**Priority:** P2.  
**Verification method:** Load testing, stress testing, capacity analysis, and architectural review.

### Scalability scenarios

| Scenario | Expected behavior |
|---|---|
| Gradual user growth | The system must support a potential 50% increase in users within less than one year without significant degradation. |
| Sudden peak at class start | The system should scale or handle the peak without losing attendance records. Proposed scaling time: < 5 minutes. |
| Load reduction | Scale-down or resource optimization must not interrupt active traffic. |
| Concurrent facial recognition requests | The system must queue or throttle requests without corrupting attendance data. |
| Horizontal scaling limit | To be defined according to the chosen infrastructure. |

### Scaling strategy

Proposed strategy:

- Stateless application services where possible.
- Session state stored in an external cache or secure token mechanism.
- Persistent data stored in the database.
- File uploads, such as justification supports, stored in object storage or a dedicated file service.
- Facial recognition comparisons should not depend on local server state.
- Background jobs may be used for reports, notifications, and audit processing.

### Scalability risks

- External facial recognition service limits.
- Database bottlenecks during entry peaks.
- File storage growth due to justification supports.
- Camera or device saturation in a single environment.
- Network congestion at the beginning of class.

---

## NFR-004: Security

**SRS source:** RNF 4 — Security.  
**Priority:** P1.  
**Verification method:** Security checklist, code review, dependency scanning, authentication tests, and data-protection validation.

### Authentication and authorization

- All private endpoints must require an authenticated user.
- Authentication must be performed before accessing protected functionality.
- **Adopted mechanism:** opaque session ids (`identity.user_session`) sent as `Authorization: Bearer <sessionId>` and validated per Java service by `AuthTokenFilter`; Kong carries no `jwt` plugin (edge JWT is the ADR-007 target state).
- Session lifetime: closed by logout (`session_status = Closed`); `password_policy.expiration_days` = 90 for passwords. There are no access/refresh tokens today — see `07-api/authentication.md`.
- Role-Based Access Control (RBAC) must be implemented.
- Roles considered in the SRS:
  - SuperAdministrator.
  - Administrator.
  - Teacher / Instructor.
  - Student.
  > Effective catalog in the database: `Administrador | Instructor | Aprendiz` (roles are global, no `school_id`).

### Authorization rules

- Students must not access administrative functions.
- Instructors must only access information related to their assigned environments.
- Administrators may manage users, roles, permissions, environments, and configuration.
- Justification approval must be restricted to authorized roles, according to the SRS.

### Password management

- Passwords must never be stored in plain text.
- **Adopted:** bcrypt with cost factor **12** and the `$2b` prefix (ADR-008, `SecurityConfig`).
- Password change must verify the user identity.
- Password recovery must use a secure, time-limited mechanism.
  > **Gap:** the Web/Mobile clients call `auth/forgot-password`, `auth/verify-code` and
  > `auth/reset-password`, but those endpoints are **not implemented server-side** (AT-005).

### Data transmission

- HTTPS is mandatory in production and pilot environments.
- TLS 1.2 or higher is required.
- HTTP is allowed only for local development.
- All API communication must be protected.

### Sensitive data

The following data must be treated as sensitive:

- Personal identification data.
- Contact information.
- Attendance records.
- Justification supports.
- Facial parameters or biometric templates.
- Audit logs related to user changes.

Requirements:

- Sensitive data must be encrypted at rest.
- Sensitive data must be encrypted in transit.
- Raw facial images should not be retained unless strictly necessary and explicitly justified.
- Facial templates or embeddings should be preferred over raw images where possible.
- Secrets, API keys, database credentials, and tokens must never be stored in source code.
- Secrets must be stored in environment variables or a secrets manager.

### Biometric data protection

Because FaceAttend EDU uses facial recognition, the system must consider:

- Explicit consent where required.
- Data minimization: collect only what is necessary.
- Purpose limitation: biometric data used only for attendance validation.
- Restricted access to biometric data.
- Audit trail for biometric parameter updates.
- Secure deletion or deactivation when a user leaves the system, subject to institutional retention rules.

### Rate limiting and abuse prevention

Proposed controls:

- Rate limit login attempts.
- Rate limit facial verification attempts.
- Lock or temporarily delay repeated failed authentication attempts.
- Monitor unusual patterns, such as multiple failed scans from the same device.

### OWASP Top 10

Code and dependencies must be reviewed against the OWASP Top 10 on each major release.

Recommended checks:

- Injection prevention.
- Broken authentication.
- Sensitive data exposure.
- Access control failures.
- Security misconfiguration.
- Vulnerable and outdated components.
- Identification and authentication failures.
- Software and data integrity failures.
- Security logging and monitoring failures.
- Server-side request forgery.

### Regulatory compliance

Applicable regulations and references:

- Colombian Law 1581 of 2012 — Habeas Data.
- GDPR principles, if applicable.
- Institutional data-protection policies.
- SRS security and privacy requirements.

---

## NFR-005: Compatibility

**SRS source:** RNF 3 — Compatibility.  
**Priority:** P1.  
**Verification method:** Compatibility matrix, manual testing, responsive design testing, and device testing.

### Browser compatibility

The web application must work correctly in the latest versions of:

- Google Chrome.
- Mozilla Firefox.
- Microsoft Edge.
- Safari.
- Brave.
- Opera / Opera GX.

### Operating system compatibility

The system must be compatible with:

- Windows.
- Linux, for web access.
- Android, latest supported version.

### Device compatibility

The system must support:

- Desktop computers.
- Laptops.
- Tablets.
- Mobile devices.
- Devices with integrated or external cameras, where facial recognition is used.

### Responsive design

- The interface must adapt to different screen sizes and resolutions.
- Critical actions, such as attendance registration and history consultation, must be usable on mobile devices.
- Buttons and input fields must be accessible without horizontal scrolling on supported mobile screens.

### Camera and hardware considerations

- The system must handle camera permission denial gracefully.
- The system must show a clear error message when the camera is unavailable.
- The system must provide an alternative registration flow when facial scanning fails.

---

## NFR-006: Usability, Accessibility, and Localization

**SRS source:** RNF 2 — Availability, interpreted as usability and accessibility.  
**Priority:** P2.  
**Verification method:** Usability testing, accessibility checklist, pilot feedback, and user satisfaction surveys.

### Usability targets

| Metric | Target | Notes |
|---|---:|---|
| Attendance registration time | < 1 minute per user after initial guidance | Proposed target. |
| Task success rate for attendance registration | ≥ 90% in pilot | Proposed target. |
| Error recovery success | Users can recover from common errors without external help | Error messages must be clear. |
| Learnability | New users should understand core flows with minimal training | Based on SRS requirement for easy learning. |
| User satisfaction | SUS score ≥ 70, if measured | Proposed target; optional for academic pilot. |

### Usability requirements

- The interface must be intuitive and easy to learn.
- Navigation must be clear for users with basic ICT skills.
- Error messages must be understandable and actionable.
- Critical states must be visible: present, absent, late, justified, pending justification.
- Users must receive confirmation after important actions, such as attendance registration or justification upload.

### Accessibility requirements

The system should follow accessibility best practices. Proposed target:

- WCAG 2.1 Level AA, where feasible.
- Sufficient color contrast.
- Readable font sizes.
- Keyboard navigation support.
- Alternative text for meaningful images.
- Avoid relying only on color to communicate states.

### Localization

- Initial language: Spanish.
- The system must support language configuration, according to SRS RF 4.2.
- Future languages may be added without redesigning the interface.
- Date and time formats must be clear for the institutional context.

---

## NFR-007: Observability

**SRS source:** Derived from RNF 1, RNF 4, RNF 5, and operational needs.  
**Priority:** P2.  
**Verification method:** Log review, monitoring dashboard, alert testing, and audit reports.

### Observability pillars

| Pillar | Requirement | Suggested tool |
|---|---|---|
| Logs | Structured JSON logs with timestamp, level, event type, user identifier when applicable, and correlation ID | Winston, Serilog, Logback, or equivalent |
| Metrics | RED metrics: Rate, Errors, Duration per endpoint | Prometheus + Grafana, or equivalent |
| Traces | End-to-end request tracing for critical flows | OpenTelemetry + Jaeger, or equivalent |
| Alerts | Alert in less than 5 minutes when a critical SLI violates its SLO | Alertmanager, email, or notification service |
| Audit | Business-level audit events for sensitive actions | Platform observability (centralized logging, no audit table) |

### Correlation ID

Each external request should generate a UUID `correlationId` propagated through logs, spans, and related transactions.

### Business metrics to monitor

Proposed metrics:

- Number of successful facial recognitions.
- Number of failed facial recognitions.
- Number of alternative registrations.
- Number of unregistered or NN user entries.
- Number of justifications submitted.
- Number of justifications approved and rejected.
- Average attendance registration time.
- Report generation time.
- Login failures.
- Device or camera errors, if IoT devices are used.

### Audit events

At minimum, the following events should be auditable:

- User login.
- Failed login attempts.
- User creation.
- User update.
- User deactivation.
- Role change.
- Permission change.
- Attendance registration.
- Alternative registration.
- Justification upload.
- Justification approval.
- Justification rejection.
- Facial parameter update.
- Administrative configuration changes.

### Privacy in logs

- Logs must not contain passwords.
- Logs must not contain raw biometric data.
- Logs must not contain full sensitive documents unless masked.
- Personal data should be minimized and protected.

---

## NFR-008: Maintainability

**SRS source:** Engineering requirement derived from project sustainability needs.  
**Priority:** P2.  
**Verification method:** Code coverage reports, code review, linters, build pipeline, and onboarding documentation.

### Maintainability targets

| Metric | Target | Notes |
|---|---:|---|
| Test coverage | ≥ 70% overall lines | Proposed academic target. |
| Test coverage for critical domain | ≥ 80% for authentication, attendance, and justification logic | Proposed target. |
| Cyclomatic complexity | ≤ 10 per function | Recommended. |
| Technical debt resolution | High-priority debt addressed within 1-2 sprints | Proposed target. |
| Onboarding time | A new developer can run the project locally in < 2 hours using documentation | Proposed target. |
| Average build time | < 5 minutes in CI | Proposed target. |
| Documentation | README, environment setup, and architecture notes must be updated | Required for academic continuity. |

### Code quality practices

Recommended practices:

- Consistent naming conventions.
- Linting and formatting.
- Code review before merging.
- Automated tests for critical flows.
- Clear separation between domain, application, and infrastructure concerns.
- Reusable error handling.
- Avoid hardcoded credentials or environment-specific values.

---

## NFR-009: Portability

**SRS source:** RNF 6 — Portability, plus deployment engineering requirements.  
**Priority:** P2.  
**Verification method:** Deployment tests, environment configuration review, and cross-browser testing.

### Application portability

- The web application must be accessible from supported modern browsers.
- The system should not depend on a specific browser plugin.
- Mobile flows must work on supported Android.
- The system must not depend on a specific client operating system for core functionality.

### Deployment portability

Proposed deployment requirements:

- Services should be deployable as Docker images.
- Environment-specific configuration must be provided through environment variables.
- No service should depend on host-specific files or local machine configuration.
- Database migrations must be repeatable.
- The system should be deployable in at least one of the following environments:
  - Docker Compose for academic staging.
  - Cloud virtual machine.
  - Kubernetes, if the team chooses cloud-native deployment.

## NFR-010: Disaster Recovery

**SRS source:** RNF 1 — Performance, recovery requirement; engineering requirement.  
**Priority:** P2.  
**Verification method:** Recovery simulation, backup restoration test, and incident documentation.

### Recovery objectives

| Scenario | RTO | RPO | Notes |
|---|---:|---:|---|
| Interface failure | < 5 minutes | 0 | Direct SRS requirement: recover without information loss. |
| Single service failure | < 5 minutes | 0 for stateless services | Restart or container recovery. |
| Primary database failure | < 15 minutes | < 5 minutes | Proposed target; depends on backup/replication strategy. |
| Availability zone loss | < 30 minutes | < 15 minutes | Proposed target if cloud infrastructure is used. |
| Full region disaster | < 4 hours | < 1 hour | Proposed advanced target; may be out of scope for academic pilot. |

### Backup requirements

Proposed backup policy:

- Database backups at least daily.
- Backup retention: minimum 7 days, subject to institutional policy.
- Restore tests at least once per academic cycle or pilot phase.
- Justification files and uploaded supports must be included in backup strategy.
- Backup restoration must be documented.

### Resilience requirements

- The system must handle temporary dependency failures gracefully.
- Users must receive clear messages when the system is unavailable.
- Attendance records must not be lost due to interface failure.
- Failed facial scans must not corrupt existing attendance data.
- Retry mechanisms must avoid duplicate attendance records.

---

## NFR-011: Data Integrity and Auditability

**SRS source:** Derived from RF 3, RF 5, RF 6, RF 7, and RNF 4.  
**Priority:** P1.  
**Verification method:** Database rules, audit log review, functional tests, and security tests.

### Data integrity requirements

| Requirement | Description |
|---|---|
| Unique attendance record | A student should not have duplicate entry or exit records for the same assigned block unless explicitly corrected. |
| Immutable attendance evidence | Original attendance records should not be silently deleted. Corrections must be traceable. |
| Justification linkage | A justification must be linked to a valid attendance, absence, or late-arrival case. |
| Role-based corrections | Only authorized roles may correct or evaluate attendance-related information. |
| Audit trail | Administrative changes to users, roles, permissions, facial parameters, and justifications must be logged. |
| Report consistency | Reports must reflect the current authorized state of attendance and justifications. |

### Audit retention

Proposed retention:

- Audit logs retained for at least the duration of the academic pilot.
- Sensitive administrative actions retained for institutional review if required.
- Audit logs must be protected against unauthorized modification.

---

## NFR priority matrix

| NFR | Category | Priority | Validated in CI? | Suggested owner |
|---|---|---|---|---|
| NFR-001 | Performance | P1 | Yes if automated load tests exist; otherwise manual evidence | Backend / QA |
| NFR-002 | Availability | P1 | Yes for health checks; monitoring evidence required | DevOps / Infrastructure |
| NFR-003 | Scalability | P2 | Manual or periodic load testing | Backend / Architecture |
| NFR-004 | Security | P1 | Yes for SAST/linting; manual security checklist required | Security / Backend |
| NFR-005 | Compatibility | P1 | Manual compatibility matrix | Frontend / QA |
| NFR-006 | Usability / Accessibility | P2 | Manual usability and accessibility testing | UX / Frontend |
| NFR-007 | Observability | P2 | Yes for basic smoke tests; monitoring in staging | Backend / DevOps |
| NFR-008 | Maintainability | P2 | Yes for coverage and linting if CI exists | Entire team |
| NFR-009 | Portability | P2 | Deployment tests | DevOps / Backend |
| NFR-010 | Disaster Recovery | P2 | Manual recovery simulation | DevOps / Database |
| NFR-011 | Data integrity and auditability | P1 | Functional and database tests | Backend / QA |

---

## Verification evidence

For each NFR, the team should store evidence in the repository or project documentation.

Suggested evidence:

| NFR | Evidence |
|---|---|
| Performance | Load test report, timing logs, or manual test sheet. |
| Availability | Monitoring screenshots, health check results, or availability log. |
| Scalability | Stress test report or capacity analysis. |
| Security | Security checklist, dependency scan report, and authentication tests. |
| Compatibility | Browser and device compatibility matrix. |
| Usability / Accessibility | Usability test notes, accessibility checklist, or pilot survey. |
| Observability | Log examples, dashboard screenshots, or alert configuration. |
| Maintainability | Coverage report, linter report, and onboarding guide. |
| Portability | Deployment instructions and environment configuration guide. |
| Disaster recovery | Backup restore test or recovery simulation report. |
| Data integrity | Test cases for duplicate records, unauthorized changes, and audit logs. |