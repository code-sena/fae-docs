# Domain Map — Bounded Contexts

> **What to fill in here:** The domain map is the central DDD (Domain-Driven Design) artifact.
> It defines the system's boundaries and how they relate to each other.
> Build it first with the team and domain experts in an Event Storming session.

## Before filling in this document: Event Storming

The FaceAttend EDU domain map describes the main business areas and system contexts. Its purpose is to organize the business model so the development team shares a common language and understands how system functions relate to one another.

This document does not describe tables, columns, or database details. It describes the business in terms of entities, responsibilities, rules, events, and contexts.

## Main Contexts

| Context | Type | Purpose | Related requirements |
|---|---|---|---|
| Security, Identity, and Access | Cross-cutting supporting subdomain | Manage users, roles, permissions, authentication, data protection, and auditing. | RF 1, RF 4.6, RF 5.2, RNF 4 |
| Academic Structure | Core subdomain | Manage environments, cohorts, courses, responsible users, schedules, and curriculum. | RF 2, RF 8 |
| Attendance Recording | System core | Record entries and exits through facial recognition or alternative methods. | RF 3 |
| Justification Management | Core subdomain | Manage supporting documents, approvals, and rejections of absences or late arrivals. | RF 6, RF 4.7 |
| Histories and Reports | Supporting subdomain | Consult histories, dashboards, and role-based report exports. | RF 5, RF 7 |
| Configuration and Preferences | Supporting subdomain | Manage alerts, languages, color palettes, and configurable user or system parameters. | RF 4 |

---

## 1. Security, Identity, and Access

### Purpose
Ensure that only authorized people can access the system and perform actions according to their roles. This context handles authentication, authorization, user management, role assignment, permissions, sensitive-data protection, and traceability of critical actions.

### Context Type
Cross-cutting supporting subdomain.

### Relationship to the Project
This domain is necessary because FaceAttend EDU handles sensitive academic information, attendance records, personal data, and facial parameters. Different roles such as Supervisor, Administrator, Teacher/Instructor, and Student must have different permissions.

### Main Concepts
- User.
- Person.
- Credentials.
- Role.
- Permission.
- Session.
- Authentication.
- Authorization.
- User status.
- Password recovery.
- Change auditing.
- Facial parameter as sensitive data.

### Responsibilities
- Register users individually or in bulk through CSV files.
- Validate sign-in.
- Recover and change passwords.
- Activate or deactivate users.
- Delete users when they no longer have an active relationship with the system.
- Assign roles to users.
- Control permissions according to the role.
- Protect personal and biometric data.
- Record relevant system actions for auditing purposes.

### Related Requirements
- RF 1: User management.
- RF 1.1: Register users.
- RF 1.1.1: Role assignment.
- RF 1.1.2: Permission assignment.
- RF 1.2: User registration through CSV.
- RF 1.3: Sign-in.
- RF 1.4: Password recovery.
- RF 1.5: Password change.
- RF 1.6: Activate or deactivate users.
- RF 1.7: Delete users.
- RF 1.8: Consult user information.
- RF 4.6: Change a user's role.
- RF 5.2: User change history.
- RNF 4: Security.

### Business Rules
- Every user must authenticate before accessing system functions.
- Every user must have a defined role.
- System permissions must depend on the assigned role.
- A user cannot have permissions greater than those corresponding to their role.
- Users with Student or Parent roles must not be able to consult other users' information.
- A user may be deleted only when they no longer have active relationships with attendance control.
- Sensitive data, such as passwords and facial parameters, must be protected according to security and privacy criteria.
- Important actions must be recorded for auditing.

### Possible Events
- `UsuarioRegistrado`
- `UsuarioActualizado`
- `UsuarioActivado`
- `UsuarioDesactivado`
- `UsuarioEliminado`
- `RolModificado`
- `SesionIniciada`
- `ContrasenaRecuperada`
- `ContrasenaCambiada`
- `ParametroFacialActualizado`
- `CambioAuditado`

### Relationship to Other Contexts
The Security, Identity, and Access context provides the foundation for other contexts to function correctly:

- **Attendance:** needs to know who the user is and whether they are active.
- **Justifications:** needs to validate who requests and who approves.
- **Reports:** needs to validate what information each role can see.
- **Academic structure:** needs to identify instructors, students, and responsible users.
- **Configuration:** needs to validate which users may modify alerts, languages, colors, or system parameters.

---

## 2. Academic Structure

### Purpose
Organize the institutional information needed for the system to relate users, groups, environments, responsible users, and schedules.

### Main Concepts
- School.
- Course.
- Cohort.
- Group.
- Environment.
- Classroom.
- Responsible user.
- Instructor.
- Teacher.
- Schedule.
- Training block.
- Curriculum.

### Responsibilities
- Register environments or classrooms.
- Register cohorts or courses.
- Assign responsible users.
- Assign cohorts to responsible users.
- Assign schedules or time blocks.
- Add or delete courses.
- View the curriculum.

### Related Requirements
- RF 2: Environment management.
- RF 2.1: Register environments/classrooms.
- RF 2.2: Register cohorts/courses.
- RF 2.3: Assign responsible users.
- RF 2.3.1: Assign cohorts.
- RF 2.3.2: Assign schedules.
- RF 8: School management.
- RF 8.1: Add or delete courses.
- RF 8.2: Subject curriculum.

### Business Rules
- A responsible user must be a valid system user.
- A cohort must exist before being associated with a responsible user.
- A schedule must not overlap another schedule assigned to the same responsible user.
- A cohort must have availability for assigning a time block.
- A course must exist to be deleted and must not exist to be added.

### Possible Events
- `EnvironmentRegistered`
- `CohortRegistered`
- `ResponsibleUserAssigned`
- `ScheduleAssigned`
- `CourseAdded`
- `CourseDeleted`
- `CurriculumConsulted`

---

## 3. Attendance Recording

### Purpose
Reliably record user entries and exits through facial recognition or alternative methods.

### Main Concepts
- Entry record.
- Exit record.
- Attendance.
- Absence.
- Late arrival.
- Facial scan.
- Facial parameter.
- Alternative record.
- NN user.
- Casual user.
- Attendance cohort.

### Responsibilities
- Record entries and exits.
- Validate facial recognition.
- Allow alternative records when facial scanning fails.
- Control entry of unregistered personnel.
- Validate that the record corresponds to the assigned schedule.

### Related Requirements
- RF 3: Generate entry and exit records.
- RF 3.1: Record entries and exits.
- RF 3.1.1: Facial recording.
- RF 3.1.2: Alternative recording.
- RF 3.2: Control entry of unregistered personnel.
- RF 3.2.1: Alternative recording for NN users.

### Business Rules
- The user must be registered in the system.
- The facial parameter must correspond to a registered user.
- A user must have a single entry record and a single exit record per schedule.
- Entry and exit times must correspond to the assigned training block.
- If facial scanning fails, an alternative record may be used.
- NN users must be classified as unregistered or casual personnel.

### Possible Events
- `AttendanceRecorded`
- `ExitRecorded`
- `LateArrivalDetected`
- `AbsenceDetected`
- `FacialScanFailed`
- `AlternativeRecordUsed`
- `NNUserRegistered`

---

## 4. Justification Management

### Purpose
Manage justifications for absences or late arrivals, allowing supporting documents to be uploaded, validity to be evaluated, and the result to be notified.

### Main Concepts
- Absence.
- Late arrival.
- Justification.
- Supporting document.
- Valid excuse.
- Approval.
- Rejection.
- Superior.
- Result notification.

### Responsibilities
- Upload justification supporting documents.
- Validate whether the supporting document applies to the absence or late arrival.
- Approve or reject the justification.
- Automatically notify the result.
- Configure which types of justification are valid.

### Related Requirements
- RF 6: Justification management.
- RF 6.1: Upload justification supporting documents.
- RF 6.2: Approval and rejection.
- RF 6.3: Automatic result notification.
- RF 4.7: Configuration of late arrivals and justifications.

### Business Rules
- The user must have a case to justify.
- The supporting document must be uploaded correctly.
- The justification must be evaluated by a superior.
- The supporting document must be valid according to criteria defined by the administrator or superior.
- The justification result must be notified to the user.

### Possible Events
- `JustificationRequested`
- `SupportingDocumentUploaded`
- `JustificationApproved`
- `JustificationRejected`
- `ResultNotified`

---

## 5. Histories and Reports

### Purpose
Enable consultation of historical attendance, absence, late-arrival, and other indicator information according to the user's role.

### Main Concepts
- History.
- Report.
- Dashboard.
- Filter.
- Date range.
- Cohort.
- Environment.
- Instructor.
- User.
- Late arrival.
- Absence.

### Responsibilities
- Consult attendance and absences.
- Filter by date, cohort, environment, instructor, or person.
- Generate reports.
- Export reports.
- View analytical dashboards.
- Consult late arrivals and absences by parameters.

### Related Requirements
- RF 5: Generate histories.
- RF 5.1: Attendance and absences.
- RF 7: Advanced visualization and reporting.
- RF 7.1: Export report.
- RF 7.2: Analytical dashboard.
- RF 7.3: Reports by date range.
- RF 7.4: Query late arrivals and absences by parameters.

### Business Rules
- The user must be registered in the system.
- Report access must depend on the user's role.
- Filters must validate that the requested information exists.
- Entered dates must be valid.
- Each role may access only permitted reports.

### Possible Events
- `HistoryConsulted`
- `ReportGenerated`
- `ReportExported`
- `DashboardViewed`

---

## 6. Configuration and Preferences

### Purpose
Manage customization options, alerts, languages, appearance, and user or system parameters.

### Main Concepts
- Alert.
- Notification.
- Language.
- Color palette.
- User parameter.
- Facial parameter.
- System configuration.

### Responsibilities
- Activate or deactivate alerts.
- Change the notification tone.
- Configure languages.
- Update color palettes.
- Update user parameters.
- Manage IoT device-failure alerts.

### Related Requirements
- RF 4: System configuration.
- RF 4.1: Configure alerts.
- RF 4.1.1: Enable/disable alerts.
- RF 4.1.2: Change notification tone.
- RF 4.2: Configure languages.
- RF 4.3: Update color palettes.
- RF 4.4: Update user parameters.
- RF 4.5: Manage IoT device-failure alerts.

### Business Rules
- The user must be signed in to modify personal settings.
- Only the administrator may modify global system settings.
- Color palettes must be configured according to the administrator's criteria.
- IoT device-failure alerts must be visible to responsible roles.

### Possible Events
- `AlertConfigured`
- `AlertDisabled`
- `LanguageUpdated`
- `ColorPaletteUpdated`
- `UserParameterUpdated`
- `IoTDeviceFailureReported`
