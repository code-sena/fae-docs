# HU-IAM-001: User login

## Story

**As** a registered user (student, instructor, administrator)  
**I want** to log in to FaceAttend EDU using my credentials  
**So that** I can securely access the modules and information allowed by my role.

---

## Acceptance criteria

> Format: “Given [context/initial state], when [user action], then [expected and verifiable result].”

- [ ] **AC1:** Given that a user is registered and active in the system, when the user submits valid credentials, then the system authenticates the user, creates a secure session or token, and grants access according to the assigned role.

- [ ] **AC2:** Given that a user submits invalid credentials, when the login request is processed, then the system denies access, shows a generic authentication error, and does not reveal whether the username exists.

- [ ] **AC3:** Given that a user account is inactive or deactivated, when the user attempts to log in, then the system denies access and shows a clear message indicating that the account is not active.

- [ ] **AC4:** Given that a user repeatedly submits invalid credentials, when the configured failed-attempt threshold is reached, then the system temporarily blocks or delays further attempts and records the event for auditing.

- [ ] **AC5:** Given that the system is running in a production or pilot environment, when the user submits login credentials, then the communication is protected using HTTPS.

- [ ] **AC6:** Given normal network conditions, when the user submits valid credentials, then the login process responds in less than 5 seconds, according to RNF 1.

- [ ] **AC7:** Given that the login is successful, when the user accesses the system, then the system records a login audit event with date, time, user identifier, and result.

---

## Technical notes

- The SRS RF 1.3 presents an inconsistency: the input is described as “Registro en FaceAttend EDU”, but the correct input should be user credentials such as username/email and password.
- Passwords must not be stored or transmitted in plain text.
- The system should implement secure password hashing, for example bcrypt or Argon2id.
- Failed login attempts should be logged without exposing sensitive data.
- Session or token expiration must be defined by the team. Proposed values: access token 1 hour, refresh token 7 days.
- The login flow must consider role-based redirection after successful authentication.
- Sensitive authentication data must not be written to logs or shown in the user interface.

**Responsible service(s):** IAM service / Authentication service, proposed.  
**Endpoint(s) implemented:**  
- `POST /api/v1/auth/login` — proposed.  
- `POST /api/v1/auth/logout` — proposed.  

**Events generated:**  
- `UserLoggedIn`  
- `LoginFailed`  
- `UserTemporarilyBlocked`  

**Required permissions:**  
- Public endpoint for login.  
- Authenticated session required after successful login.

---

## Definition of Done (DoD)

> This HU can only be closed when it meets the team's full DoD.

**Additional checks specific to this HU:**

- [ ] Valid login scenario tested.
- [ ] Invalid login scenario tested.
- [ ] Inactive user scenario tested.
- [ ] Repeated failed attempts scenario tested.
- [ ] Password is not exposed in logs, responses, or UI.
- [ ] HTTPS is enforced in production/pilot.
- [ ] Login audit event is generated.
- [ ] Error messages are clear but do not reveal sensitive information.
- [ ] Login response time is validated under normal conditions and remains under 5 seconds.
- [ ] Traceability to RF 1.3 is documented.

---

## Estimation and priority

| Field | Value |
|---|---|
| Story Points | 5 |
| Priority | High |
| Target sprint | Sprint 1 / MVP |
| Dependencies | HU-IAM-002: User registration, or an initial seeded administrator user |

---

# HU-IAM-002: User registration with role assignment

## Story

**As** an administrator or authorized institutional user  
**I want** to register a new user and assign a role  
**So that** the user can exist in the system and access the modules corresponding to their institutional function.

---

## Acceptance criteria

> Format: “Given [context/initial state], when [user action], then [expected and verifiable result].”

- [ ] **AC1:** Given that an administrator provides valid user data, when the user registration is submitted, then the system validates that the user does not already exist, creates the user, assigns the selected role, stores the data securely, and shows a confirmation message.

- [ ] **AC2:** Given that the user data is incomplete or invalid, when the registration is submitted, then the system rejects the operation and shows specific validation errors for the missing or incorrect fields.

- [ ] **AC3:** Given that a user already exists with the same institutional email, identifier, or username, when the administrator attempts to register a new user, then the system prevents duplicate registration and shows a clear error message.

- [ ] **AC4:** Given that a role is selected during registration, when the user is created, then the system associates the user identifier with the person identifier and the assigned role, according to RF 1.1.1.

- [ ] **AC5:** Given that the selected role does not exist or is not allowed for the new user, when the registration is submitted, then the system rejects the operation and shows a validation error.

- [ ] **AC6:** Given that the registration is successful, when the administrator consults the audit or history, then the system records the events `UserRegistered` and `RoleAssigned`.

- [ ] **AC7:** Given normal network conditions, when the administrator completes user registration, then the system responds in less than 5 seconds, according to RNF 1.

---

## Technical notes

- This story covers basic user registration and role assignment. Massive user import by CSV is covered separately in HU-IAM-004.
- The system must validate that the user does not exist before registration, according to RF 1.1 acceptance criteria.
- The system must validate the entered data, according to RF 1.1 acceptance criteria.
- Role assignment must include the relationship between user identifier and person identifier, according to RF 1.1.1.
- User roles considered in the SRS include:
  - Supervisor.
  - Administrator.
  - Teacher / Instructor.
  - Student.
  - Parent or guardian, if applicable.
- Password handling must follow security requirements. If registration generates an initial password or invitation flow, this must be defined by the team.
- Sensitive data must be stored securely and must not appear in logs.
- Registration should be auditable for traceability.

**Responsible service(s):** IAM service / User management service, proposed.  
**Endpoint(s) implemented:**  
- `POST /api/v1/users` — proposed.  
- `POST /api/v1/users/{userId}/role` — proposed, if role assignment is separated from user creation.  

**Events generated:**  
- `UserRegistered`  
- `RoleAssigned`  
- `UserRegistrationFailed`  

**Required permissions:**  
- Administrator or Supervisor, depending on institutional policy.  
- Self-registration should only be allowed if explicitly approved by the institution.

---

## Definition of Done (DoD)

> This HU can only be closed when it meets the team's full DoD.

**Additional checks specific to this HU:**

- [ ] Successful user registration tested.
- [ ] Invalid or incomplete data scenario tested.
- [ ] Duplicate user scenario tested.
- [ ] Role assignment tested.
- [ ] Invalid role scenario tested.
- [ ] User-person-role relationship is correctly stored.
- [ ] Sensitive data is not exposed in logs or UI.
- [ ] Audit events are generated.
- [ ] Confirmation and error messages are clear.
- [ ] Response time is validated under normal conditions and remains under 5 seconds.
- [ ] Traceability to RF 1.1 and RF 1.1.1 is documented.

---

## Estimation and priority

| Field | Value |
|---|---|
| Story Points | 8 |
| Priority | High |
| Target sprint | Sprint 1 / MVP |
| Dependencies | Initial role catalog; HU-IAM-003 recommended for permission matrix |

---

# HU-IAM-003: Role-based permission assignment

## Story

**As** an administrator  
**I want** to assign permissions to each role  
**So that** users can only access the modules, views, and actions allowed by their active role.

---

## Acceptance criteria

> Format: “Given [context/initial state], when [user action], then [expected and verifiable result].”

- [ ] **AC1:** Given that an administrator is authenticated, when the administrator assigns permissions to a role, then the system stores the role-permission relationship and applies it to users assigned to that role.

- [ ] **AC2:** Given that a user logs in with an assigned role, when the system loads the user interface or API permissions, then the user only sees and can access the modules and actions allowed for that role.

- [ ] **AC3:** Given that a user attempts to access a module, view, or action not allowed by their role, when the request is processed, then the system denies access and records the unauthorized attempt.

- [ ] **AC4:** Given that an administrator updates the permissions of a role, when the changes are saved, then the new permissions are applied to subsequent sessions or token validations and an audit event is generated.

- [ ] **AC5:** Given that the administrator tries to assign an invalid module, view, or action, when the permission assignment is submitted, then the system rejects the operation and shows a validation error.

- [ ] **AC6:** Given that permissions have been assigned to roles, when an administrator consults the permission matrix, then the system displays the permissions associated with each role.

- [ ] **AC7:** Given normal network conditions, when the administrator consults or updates role permissions, then the system responds in less than 5 seconds, according to RNF 1.

---

## Technical notes

- This story implements the foundation for Role-Based Access Control, according to RF 1.1.2.
- Permission validation must be enforced on the server side. Client-side hiding of options is not sufficient.
- The system should support a role-permission matrix or equivalent authorization model.
- Roles considered in the SRS include:
  - Supervisor.
  - Administrator.
  - Teacher / Instructor.
  - Student.
  - Parent or guardian, if applicable.
- The permission model should consider modules, views, and actions if the system requires fine-grained access control.
- Unauthorized access attempts should be logged for auditing purposes.
- Permission changes must be traceable and should appear in user-change history or audit logs.
- The system must avoid privilege escalation, meaning users should not be able to grant themselves permissions.

**Responsible service(s):** IAM service / Authorization service, proposed.  
**Endpoint(s) implemented:**  
- `GET /api/v1/roles` — proposed.  
- `GET /api/v1/roles/{roleId}/permissions` — proposed.  
- `POST /api/v1/roles/{roleId}/permissions` — proposed.  
- `GET /api/v1/users/me/permissions` — proposed.  

**Events generated:**  
- `RolePermissionsUpdated`  
- `UnauthorizedAccessAttempt`  

**Required permissions:**  
- Administrator or Supervisor for role-permission management.  
- Regular users may only read their own effective permissions.

---

## Definition of Done (DoD)

> This HU can only be closed when it meets the team's full DoD.

**Additional checks specific to this HU:**

- [ ] Permission assignment by role tested.
- [ ] Permission update tested.
- [ ] Unauthorized access attempt tested.
- [ ] Permission matrix consultation tested.
- [ ] Server-side authorization validation tested.
- [ ] Audit event for permission changes tested.
- [ ] Invalid permission assignment scenario tested.
- [ ] Sensitive authorization data is not exposed in logs.
- [ ] Response time is validated under normal conditions and remains under 5 seconds.
- [ ] Traceability to RF 1.1.2 is documented.

---

## Estimation and priority

| Field | Value |
|---|---|
| Story Points | 5 |
| Priority | High |
| Target sprint | Sprint 1 / MVP |
| Dependencies | HU-IAM-002: User registration with role assignment |