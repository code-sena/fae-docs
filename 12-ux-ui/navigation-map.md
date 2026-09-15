# Navigation Map

> Defines the screen structure of the FaceAttend EDU system, how screens connect to each other,
> and what routes exist. It is the reference when frontend and backend discuss what endpoints
> exist or how to reach a feature.
>
> **Platforms:** Web (admin-only) + Mobile (learners and instructors).
> **Roles:** Administrator, Instructor, Learner (Student).

---

## Frontend route structure

> Web application is exclusively for Administrators.
> Mobile application is for Learners and Instructors with accounts created by the Administrator.

### Web routes (Admin only)

```
/                               → Landing page (public, no auth required)
│
├── /admin/login                → Administrator authentication
│
├── /admin/dashboard            → Admin main panel (authenticated)
│   ├── /admin/users            → User management (view, edit, delete learners & instructors)
│   ├── /admin/environments     → Environment/classroom management
│   ├── /admin/fichas           → Ficha and course management
│   ├── /admin/instructors      → Instructor assignment by ficha, environment, day, and course
│   ├── /admin/school-info      → School information management
│   ├── /admin/reports          → Reports (by user, environment, ficha; filters by date & program; exportable)
│   └── /admin/scan-face        → Standalone face scan section (no additional sub-sections)
│
└── /admin/settings             → System configuration (if applicable)
```

### Mobile routes (Learners & Instructors)

```
/login                          → Login (first screen, mobile only)
│
├── /password-reset             → Password recovery (mobile only)
│
├── /dashboard                  → Main panel (shows role and user name)
│
├── /attendance
│   └── /register               → Face/fingerprint attendance registration (dedicated screen)
│
├── /face
│   ├── /register               → Initial face registration for database
│   └── /update                 → Face parameter update flow
│       ├── /update/verify-identity   → Step 1: Fill user data to validate identity
│       ├── /update/scan-face         → Step 2: Scan and submit face
│       └── /update/validate          → Step 3: Validate if face photo is correct
│
├── /justifications
│   ├── /                       → Justifications menu
│   ├── /submitted              → View submitted justifications
│   ├── /results                → View approved/rejected justifications
│   └── /submit                 → Submit new justification (date, instructor, environment, reason, attachments)
│
├── /history                    → Attendance history (view only)
│
├── /school-info                → School information (view only, no edit)
│
├── /academic                   → Academic section (view only)
│
├── /contacts                   → Contacts section
│
└── /settings                   → Settings (single screen)
    ├── Language selection
    ├── Theme toggle (light/dark)
    └── Alert configuration (enable/disable alerts)
```

---

## Screen map

### Web screens (Administrator)

| Screen | Route | Component | Minimum role | Backend service |
|--------|-------|-----------|--------------|----------------|
| Landing Page | `/` | `LandingPage` | Public | — |
| Admin Login | `/admin/login` | `AdminLoginPage` | Public | user-auth-service |
| Admin Dashboard | `/admin/dashboard` | `AdminDashboardPage` | ADMIN | roles-permissions-service |
| User Management | `/admin/users` | `UserManagementPage` | ADMIN | user-auth-service |
| Environment Management | `/admin/environments` | `EnvironmentManagementPage` | ADMIN | academic-service |
| Ficha & Course Management | `/admin/fichas` | `FichaCourseManagementPage` | ADMIN | academic-service |
| Instructor Assignment | `/admin/instructors` | `InstructorAssignmentPage` | ADMIN | schedule-service |
| School Info Management | `/admin/school-info` | `SchoolInfoManagementPage` | ADMIN | academic-service |
| Reports | `/admin/reports` | `ReportsPage` | ADMIN | attendance-service |
| Face Scan (standalone) | `/admin/scan-face` | `FaceScanPage` | ADMIN | biometric-service |

### Mobile screens (Learners & Instructors)

| Screen | Route | Component | Minimum role | Backend service |
|--------|-------|-----------|--------------|----------------|
| Login | `/login` | `LoginPage` | Public | user-auth-service |
| Password Reset | `/password-reset` | `PasswordResetPage` | Public | user-auth-service |
| Dashboard | `/dashboard` | `DashboardPage` | LEARNER / INSTRUCTOR | roles-permissions-service |
| Attendance Registration | `/attendance/register` | `AttendanceRegisterPage` | LEARNER / INSTRUCTOR | attendance-service, biometric-service |
| Face Registration | `/face/register` | `FaceRegisterPage` | LEARNER / INSTRUCTOR | biometric-service |
| Face Update - Verify Identity | `/face/update/verify-identity` | `FaceUpdateVerifyPage` | LEARNER / INSTRUCTOR | biometric-service |
| Face Update - Scan Face | `/face/update/scan-face` | `FaceUpdateScanPage` | LEARNER / INSTRUCTOR | biometric-service |
| Face Update - Validate | `/face/update/validate` | `FaceUpdateValidatePage` | LEARNER / INSTRUCTOR | biometric-service |
| Justifications Menu | `/justifications` | `JustificationsMenuPage` | LEARNER / INSTRUCTOR | attendance-service |
| Submitted Justifications | `/justifications/submitted` | `JustificationsSubmittedPage` | LEARNER / INSTRUCTOR | attendance-service |
| Justification Results | `/justifications/results` | `JustificationsResultsPage` | LEARNER / INSTRUCTOR | attendance-service |
| Submit Justification | `/justifications/submit` | `JustificationSubmitPage` | LEARNER | attendance-service |
| Attendance History | `/history` | `AttendanceHistoryPage` | LEARNER / INSTRUCTOR | attendance-service |
| School Info (view only) | `/school-info` | `SchoolInfoPage` | LEARNER / INSTRUCTOR | academic-service |
| Academic Section | `/academic` | `AcademicPage` | LEARNER / INSTRUCTOR | academic-service |
| Contacts | `/contacts` | `ContactsPage` | LEARNER / INSTRUCTOR | academic-service |
| Settings | `/settings` | `SettingsPage` | LEARNER / INSTRUCTOR | config-integration-service |

### Role-based access summary

| Screen / Module | Learner | Instructor | Admin (Web) |
|----------------|:-------:|:----------:|:-----------:|
| View own attendance | ✅ | ✅ | ✅ |
| Register face / fingerprint | ✅ | ✅ | — |
| Update face parameters | ✅ | ✅ | — |
| Submit justifications | ✅ | ✅ | — |
| Approve/deny justifications | ❌ | ✅ | ✅ |
| Change theme / language / alerts | ✅ | ✅ | — |
| View school info (read-only) | ✅ | ✅ | ✅ |
| View academic section | ✅ | ✅ | ✅ |
| View contacts | ✅ | ✅ | ✅ |
| Manage users (CRUD) | ❌ | ❌ | ✅ |
| Manage environments | ❌ | ❌ | ✅ |
| Manage fichas & courses | ❌ | ❌ | ✅ |
| Assign instructors | ❌ | ❌ | ✅ |
| Manage school info (edit) | ❌ | ❌ | ✅ |
| View/export reports | ❌ | ❌ | ✅ |
| Face scan (standalone) | ❌ | ❌ | ✅ |

---

## Main user flows

### Flow 1 — Authentication (Mobile Login)

```
Login (/login)
    │
    ▼ User enters credentials
    │
    ├── Valid credentials ──────────► Dashboard (/dashboard)
    │                                  Shows: role + user name
    │
    ├── Invalid credentials ────────► Login with error message
    │                                  (max. 3 attempts, then blocked)
    │
    └── Deactivated user ───────────► Login denied
                                       "Invalid credentials" message
```

**Related HUs:** HU-IAM-001, HU-IAM-007

**Notes:**
- Maximum 3 login attempts allowed.
- Deactivated learners/instructors cannot pass login; they receive "invalid credentials" message.
- Upon successful login, the dashboard displays the user's role and name immediately.
- No dedicated "authentication success" screen exists; the user goes directly to the dashboard.

---

### Flow 2 — Facial / Fingerprint Attendance Registration (Mobile)

```
Dashboard (/dashboard)
    │
    ▼ Navigate to attendance registration
    │
Attendance Registration (/attendance/register)
    │
    ▼ User initiates face scan
    │
    ├── Face recognized ────────────► Attendance registered successfully
    │                                  Confirmation message shown
    │
    ├── Face NOT recognized ────────► Option to use fingerprint
    │   │
    │   ├── Fingerprint matches ────► Attendance registered successfully
    │   │
    │   └── Fingerprint fails ──────► Error message / retry option
    │
    └── Camera failure ─────────────► Option to use fingerprint
        │
        ├── Fingerprint matches ────► Attendance registered successfully
        │
        └── Fingerprint fails ──────► Error message / retry option
```

**Related HUs:** HU-ATT-001, HU-ATT-003

**Notes:**
- Attendance registration is done from a dedicated screen, not from the dashboard directly.
- If facial recognition fails (no match, camera failure), the user can manually select fingerprint as the alternative method.
- The alternative registration flow has no fixed time limit and does not expire.

---

### Flow 3 — Justification Submission (Mobile - Learner)

```
Login (/login)
    │
    ▼ Successful authentication
    │
Dashboard (/dashboard)
    │
    ▼ Navigate to app menu
    │
Justifications Menu (/justifications)
    │
    ▼ Select "Justifications to submit"
    │
Submit Justification (/justifications/submit)
    │
    ▼ Fill in required data:
    │   - Date
    │   - Instructor of the day
    │   - Environment
    │   - Reason
    │   - Required attachments
    │
    ▼ Submit
    │
    ├── Successful submission ──────► Confirmation message
    │                                  Justification status: Pending
    │
    └── Submission error ───────────► Error message with details
```

**Related HUs:** HU-JUS-001, HU-JUS-002, HU-JUS-003

**Notes:**
- Only Learners can submit justifications.
- Instructors can approve or deny justifications submitted by learners.
- The administrator can also approve or deny justifications.

---

### Flow 4 — Justification Evaluation (Mobile - Instructor / Admin)

```
Dashboard (/dashboard)
    │
    ▼ Navigate to justifications
    │
Justifications Menu (/justifications)
    │
    ▼ View submitted justifications
    │
Submitted Justifications (/justifications/submitted)
    │
    ▼ Select a justification to evaluate
    │
    ├── Approve ────────────────────► Status updated to "Approved"
    │                                  Notification sent to learner
    │
    └── Deny ───────────────────────► Status updated to "Denied"
                                       Notification sent to learner
```

**Related HUs:** HU-JUS-002, HU-JUS-003

---

### Flow 5 — Face Parameter Update (Mobile)

```
Dashboard (/dashboard)
    │
    ▼ Navigate to face update
    │
Step 1: Verify Identity (/face/update/verify-identity)
    │
    ▼ Fill user data to validate identity
    │
    ├── Identity verified ──────────► Step 2
    │
    └── Identity NOT verified ──────► Error message / retry
    │
Step 2: Scan Face (/face/update/scan-face)
    │
    ▼ Scan and submit new face image
    │
Step 3: Validate (/face/update/validate)
    │
    ├── Photo is correct ───────────► Face parameters updated successfully
    │                                  Confirmation message shown
    │
    └── Photo is NOT correct ───────► Option to retry scan
```

**Related HUs:** HU-CONF-006, HU-ATT-002

---

### Flow 6 — Admin User Management (Web)

```
Admin Dashboard (/admin/dashboard)
    │
    ▼ Navigate to admin panel section
    │
User Management (/admin/users)
    │
    ├── View all users ─────────────► List of learners and instructors
    │
    ├── Edit user ──────────────────► Edit form → Confirmation dialog → Save
    │
    ├── Delete user ────────────────► Confirmation dialog → Delete
    │
    └── Create user ────────────────► Creation form → Validate → Save
```

**Related HUs:** HU-IAM-002, HU-IAM-007, HU-IAM-008, HU-IAM-009

**Notes:**
- The administrator has a dedicated section to access all management areas quickly.
- Destructive actions (delete, edit critical data) require a confirmation dialog before executing.

---

### Flow 7 — Report Generation and Export (Web - Admin)

```
Admin Dashboard (/admin/dashboard)
    │
    ▼ Navigate to reports
    │
Reports (/admin/reports)
    │
    ▼ Select report type:
    │   - By user
    │   - By environment
    │   - By ficha
    │   - By instructor
    │
    ▼ Apply filters:
    │   - Date range
    │   - Program
    │
    ├── View report ────────────────► Report displayed on screen
    │
    └── Export report ──────────────► File downloaded
```

**Related HUs:** HU-REP-001, HU-REP-003, HU-REP-004

---

## Navigation rules

| Rule | Description |
|------|-------------|
| Authentication required | All routes except the landing page (`/`) and login screens require an active session. Users cannot navigate beyond the login screen without authenticating. |
| Login attempt limit | Maximum of 3 failed login attempts. After 3 failures, the account is temporarily blocked. |
| Deactivated users | Deactivated users cannot log in. They receive an "invalid credentials" message without revealing the account is deactivated. |
| Role-based access | Routes and screens are restricted by role. Learners and Instructors access mobile routes. Administrators access web routes. |
| Authorization (403) | If a user with insufficient permissions attempts to access a restricted route, the system displays a 403 error. No automatic redirect to dashboard is performed. |
| 404 handling | The system does not implement custom 404 error handling. |
| Confirmation dialogs | All destructive or critical actions (delete, edit important information) require a confirmation dialog before execution. |
| Web platform scope | The web platform is exclusively for Administrators. It includes user management, academic management, reports, school info, and a standalone face scan section. |
| Mobile platform scope | The mobile platform is exclusively for Learners and Instructors with accounts previously created by the Administrator. |
| Justification permissions | Only Learners can submit justifications. Instructors and Administrators can approve or deny them. |
| School info permissions | Learners and Instructors can view school information in read-only mode. Only Administrators can edit it. |

---

## Backend services reference

| Service | Description |
|---------|-------------|
| `user-auth-service` | User registration, authentication, login, password recovery, account activation/deactivation. |
| `roles-permissions-service` | Role assignment, permission management, access control validation. |
| `academic-service` | Environment management, ficha/course management, school information, academic structure. |
| `schedule-service` | Instructor assignment by ficha, environment, day, and course. Schedule and class management. |
| `attendance-service` | Attendance registration (entry/exit), attendance history, justification management, reports. |
| `biometric-service` | Facial recognition, fingerprint recognition, face parameter registration and update. |
| `alerts-service` | Alert configuration, notification dispatch, alert enable/disable. |
| `audit-service` | Audit logging, error tracking, system event recording. |
| `config-integration-service` | Language configuration, theme settings, external service integration. |

---

## Correlations

- User Stories → `04-requirements/user-stories.md`
- Non-Functional Requirements → `04-requirements/non-functional.md`
- Traceability Matrix → `04-requirements/traceability-matrix.md`
- Domain Entities and Rules → `02-domain/entities-and-rules.md`
- Roles and Permissions → `00-governance/security-policy.md` *(if applicable)*  