# Endpoint Matrix — FaceAttendEDU

> Cross-service canonical table. `Kong` = path via `:8080`; `Direct` = `http://localhost:<port>`.
> `Auth`: `public` (no session required) · `session` (Bearer opaque `sessionId`) ·
> `—` (route not implemented).
>
> **Kong authenticates at the edge.** A global `pre-function` (Lua) requires
> `Authorization: Bearer <uuid>` on every non-public route and introspects
> `GET ms-identity:8081/api/v1/sessions/{id}` accepting only `sessionStatus = "Active"`:
> **401** when the header is missing, the token is not an 8-4-4-4-12 UUID or the session is
> invalid/expired, **503** when identity does not answer. There is no cache — every authenticated
> request costs one call to identity, and the container needs
> `KONG_UNTRUSTED_LUA_SANDBOX_REQUIRES=resty.http`. The bearer is an **opaque session id, not a
> JWT**; `jwt` RS256 issued by identity stays a future evolution (ADR-007) and Kong carries no
> `jwt` plugin nor consumers. The gateway **only authenticates**: permissions (RBAC) are
> evaluated by every service (see §11). Full rules and public-route
> list in `authentication.md`; details per contract in `contracts/openapi/`.

> **Legacy alias (not a contract):** the Java services also answer paths **without** the
> `/api/v1` prefix — `/roles`, `/permissions`, `/user-roles`, `/class-sessions`,
> `/schedule-blocks`, `/environments`, `/attendance-records` (`+ /bulk`, `+ /summary`),
> `/justifications`, `/justification-types`, `/supporting-documents`, … — declared as a second
> value in the same `@RequestMapping`/`@GetMapping`. Kong routes only `/api/v1/...`, so the alias
> is reachable on the direct port only, and the MS-side `AuthTokenFilter` skips it (it guards
> paths that start with `/api/`). Treat it as a legacy accident, never as an API surface.

## 1. Identity 01 (`01-ms-identity`, Java 8081 → `01-identity-service.yaml`)

| METHOD path | Kong | Auth | Description |
|-------------|------|------|-------------|
| POST /api/v1/auth/login | ✅ | public | Login (`identifier` = email **or** username) → `{sessionId,userId,sessionStatus}` |
| POST /api/v1/auth/refresh | ✅ | public | Rotate the opaque session → new `UserSessionDto`, old one closed (401 if expired or already rotated) |
| POST /api/v1/auth/logout?sessionId | ✅ | public | Logout → 204 |
| GET /api/v1/auth/me?username\|email | ✅ | public | Current user → `UserDto` (params optional) |
| POST /api/v1/auth/forgot-password | ✅ | public | Issue + email a 6-digit code → 202 `{status:"accepted"}` (anti-enumeration) |
| POST /api/v1/auth/verify-code | ✅ | public | Validate `{email,code}` (6 digits, 10 min, 5 attempts, 60 s resend cooldown) → 200 `{status:"verified"}` |
| POST /api/v1/auth/change-password | ✅ | session | `{currentPassword,newPassword}` → 200 `{status:"updated"}`; closes the other sessions |
| POST /api/v1/auth/reset-password | ✅ | public | `{email,password}` after a verified challenge → 200 `{status:"updated"}` |
| POST /api/v1/users | ✅ | session | Create user → 201 |
| GET /api/v1/users?page&limit | ✅ | session | List → `{data,meta{page,limit,total,totalPages}}` |
| GET /api/v1/users/{id} | ✅ | session | Get user |
| PUT /api/v1/users/{id} | ✅ | session | Update → 204 |
| POST /api/v1/users/{id}/activate | ✅ nested | session | Activate → 204 |
| PATCH /api/v1/users/{id}/status | ✅ | session | Status → 204 |
| POST /api/v1/persons | ✅ | session | Create person |
| GET /api/v1/persons?page&limit | ✅ | session | List → `{data,meta}` |
| GET /api/v1/persons/{id} | ✅ | session | Get person |
| PUT /api/v1/persons/{id} | ✅ | session | Update → 204 |
| POST /api/v1/persons/{id}/transfer | ❌ not implemented | — | Person has no `school_id`; use `academic_actor` |
| PATCH /api/v1/persons/{id}/status | ✅ | session | Status → 204 |
| POST /api/v1/cities | ✅ | session | Legacy — model deprecated (cities from external `countriesnow.space`); table kept by Liquibase |
| GET /api/v1/cities?page&limit | ✅ | session | Legacy → `{data,meta}` (see external API) |
| GET /api/v1/cities/{id} | ✅ | session | Legacy — see external API |
| PUT /api/v1/cities/{id} | ✅ | session | Legacy — see external API |
| DELETE /api/v1/cities/{id} | ✅ | session | Legacy — see external API |
| POST /api/v1/password-policies | ✅ | session | Create policy → 201 |
| GET /api/v1/password-policies | ✅ | session | List |
| GET /api/v1/password-policies/{id} | ✅ | session | Get |
| PUT /api/v1/password-policies/{id} | ✅ | session | Update → 204 |
| POST /api/v1/sessions?userId&sourceIp | ✅ | session | Create session → 201 |
| GET /api/v1/sessions?page&limit | ✅ | session | List → `{data,meta}` |
| GET /api/v1/sessions/{id} | ✅ | session | Get by UUID — session lookup used by the Kong `pre-function` and every `AuthTokenFilter` (identity itself permits the path so introspection works) |
| DELETE /api/v1/sessions/{id} | ✅ | session | Close → 204 |
| GET /api/v1/sessions/user/{userId} | ✅ | session | By user (identity permits it without a token; the edge still requires one) |
| GET /health, /api/v1/health | direct | public | Health |

## 2. Authorization 02 (`02-ms-authorization`, Java 8082 → `02-authorization-service.yaml`)

| METHOD path | Kong | Auth | Description |
|-------------|------|------|-------------|
| GET /api/v1/roles | ✅ | session | List |
| POST /api/v1/roles | ✅ | session | Create → 201 |
| GET/PUT/DELETE /api/v1/roles/{id} | ✅ | session | CRUD |
| GET /api/v1/roles/{roleId}/permissions | ✅ | session | Role permissions |
| POST /api/v1/roles/{roleId}/permissions | ✅ | session | Assign → 201 |
| DELETE /api/v1/roles/{roleId}/permissions/{permId} | ✅ | session | Remove → 204 |
| GET /api/v1/permissions | ✅ | session | List |
| POST /api/v1/permissions | ✅ | session | Create → 201 |
| GET/PUT/DELETE /api/v1/permissions/{id} | ✅ | session | CRUD |
| POST /api/v1/users/{userId}/roles | ✅ nested | session | Assign role → 201 |
| DELETE /api/v1/users/{userId}/roles/{roleId} | ✅ | session | Remove → 204 |
| GET /api/v1/users/{userId}/roles | ✅ | session | User roles |
| GET /api/v1/user-roles?userIds | ✅ | session | Batch roles → `{userId: RoleResponse[]}` (0 or missing ids → `{}`; max 300 ids) |
| GET /api/v1/auth/evaluate?userId&permission | ✅ | session | RBAC → `{allowed}` |

## 3. Academic 03 (`03-ms-academic`, TS 8083 → `03-academic-service.yaml`)

| METHOD path | Kong | Auth | Description |
|-------------|------|------|-------------|
| POST/GET /api/v1/schools, GET/PUT/DELETE /api/v1/schools/{id} | ✅ | session | CRUD + `?limit&offset` + `x-total-count` |
| PATCH /api/v1/schools/{id}/status | ✅ | session | Status |
| POST /api/v1/schools/{schoolId}/programs, GET …/programs | ✅ nested | session | By school |
| POST/GET /api/v1/programs, GET/PUT/DELETE /api/v1/programs/{id} | ✅ | session | CRUD |
| POST/GET /api/v1/academic-periods (+alias /periods), GET/PUT/DELETE /{id} | ✅ | session | CRUD per alias |
| POST/GET /api/v1/schools/{schoolId}/periods | ✅ nested | session | By school |
| POST/GET /api/v1/cohorts, GET/PUT/DELETE /api/v1/cohorts/{id} | ✅ | session | CRUD |
| GET /api/v1/programs/{programId}/cohorts | ✅ nested | session | By program |
| POST /api/v1/programs/{programId}/courses, POST/GET /api/v1/courses, GET/PUT/DELETE /{id} | ✅ | session | CRUD + by program |
| GET /api/v1/programs/{programId}/courses | ✅ nested | session | By program |
| POST/GET /api/v1/actor-types, GET/PUT/DELETE /{id} | ✅ | session | CRUD |
| POST/GET /api/v1/academic-actors, GET/PUT/DELETE /{id} | ✅ | session | CRUD |
| PATCH /api/v1/academic-actors/{id}/status | ✅ | session | Status |
| GET /api/v1/schools/{schoolId}/actors | ✅ nested | session | By school |
| POST/GET /api/v1/enrollments, GET/PUT/DELETE /{id} | ✅ | session | CRUD |
| PATCH /api/v1/enrollments/{id}/status | ✅ | session | `Active\|Withdrawn\|Completed` |
| GET /api/v1/cohorts/{cohortId}/enrollments | ✅ nested | session | By cohort |
| GET /api/v1/academic-actors/{actorId}/enrollments | ✅ nested | session | By actor |

## 4. Scheduling 04 (`04-ms-scheduling`, Java 8084 → `04-scheduling-service.yaml`)

| METHOD path | Kong | Auth | Description |
|-------------|------|------|-------------|
| GET /api/v1/environments?schoolId | ✅ | session | List filtered |
| POST /api/v1/environments | ✅ | session | Create → 201 |
| GET/PUT/DELETE /api/v1/environments/{id} | ✅ | session | CRUD |
| GET /api/v1/schedule-blocks?cohortId&environmentId&instructorActorId&courseId | ✅ | session | List filtered |
| POST /api/v1/schedule-blocks | ✅ | session | Create → 201 (409 on double-booking) |
| GET/PUT/DELETE /api/v1/schedule-blocks/{id} | ✅ | session | CRUD |
| GET /api/v1/class-sessions?scheduleBlockId | ✅ | session | List filtered |
| POST /api/v1/class-sessions | ✅ | session | Create → 201 |
| GET/PUT/DELETE /api/v1/class-sessions/{id} | ✅ | session | CRUD |
| POST /api/v1/class-sessions/{id}/open\|close\|cancel | ✅ | session | Transitions (real POST) |
| GET /api/v1/cohorts/{id}/blocks | ✅ nested | session | By cohort |
| GET /api/v1/environments/{id}/blocks | ✅ | session | By environment |
| GET /api/v1/actors/{id}/blocks | ✅ | session | By instructor |
| GET /api/v1/blocks/{id}/sessions | ✅ | session | Block sessions |

## 5. Attendance 05 (`05-ms-attendance`, Java 8085 → `05-attendance-service.yaml`)

| METHOD path | Kong | Auth | Description |
|-------------|------|------|-------------|
| GET /api/v1/attendance-records?classSessionId&academicActorId&attendanceStatus | ✅ | session | List filtered |
| POST /api/v1/attendance-records | ✅ | session | Create → 201 |
| GET /api/v1/attendance-records/summary?academicActorIds | ✅ | session | Per-actor counts → `{actorId:{present,late,absent,justified,total}}` (missing ids → `{}`; max 300 ids, aggregated in SQL) |
| POST /api/v1/attendance-records/bulk | ✅ | session | Bulk → 201 |
| GET/PUT/DELETE /api/v1/attendance-records/{id} | ✅ | session | CRUD |
| GET /api/v1/justifications?status&attendanceRecordId | ✅ | session | List filtered |
| POST /api/v1/justifications | ✅ | session | Create → 201 |
| GET/DELETE /api/v1/justifications/{id} | ✅ | session | Read/Delete |
| PATCH /api/v1/justifications/{id}/review | ✅ | session | Review |
| GET /api/v1/justification-types | ✅ | session | List (`?schoolId` per-school, NULL = global) |
| POST /api/v1/justification-types | ✅ | session | Create → 201 |
| GET/PUT/DELETE /api/v1/justification-types/{id} | ✅ | session | CRUD |
| POST/GET /api/v1/attendance-reports | ❌ not implemented | — | Planned — `attendance_report` derived projection owned by 05-attendance, to implement |
| GET /api/v1/supporting-documents?justificationId | ✅ | session | List filtered |
| POST /api/v1/supporting-documents | ✅ | session | Create → 201 |
| GET/DELETE /api/v1/supporting-documents/{id} | ✅ | session | Read/Delete |
| GET /api/v1/class-sessions/{id}/attendance | ✅ nested | session | By session |
| GET /api/v1/academic-actors/{actorId}/attendance | ✅ nested | session | By actor |
| GET /api/v1/attendance/{id}/justification | ✅ nested | session | Single\|null |
| GET /api/v1/justifications/{id}/documents | ✅ nested | session | By justification |

## 6. Biometric 06 (`06-ms-biometric`, Python 8086 → `06-biometric-service.yaml`)

| METHOD path | Kong | Auth | Description |
|-------------|------|------|-------------|
| POST /api/v1/biometric/facial/enroll | ✅ | session | Enroll → 201 (0.85 threshold) |
| GET /api/v1/biometric/facial/{person_id} | ✅ | session | Active / 404 |
| GET /api/v1/biometric/facial/{person_id}/history | ✅ | session | Versions |
| DELETE /api/v1/biometric/facial/{person_id} | ✅ | session | Deactivate → 204 |
| POST /api/v1/biometric/facial/verify | ✅ | session | 1:1 `{match,score}` |
| POST /api/v1/biometric/facial/identify | ✅ | session | 1:N / 404 |
| POST /api/v1/biometric/fingerprint/enroll | ✅ | session | Enroll → 201 |
| GET /api/v1/biometric/fingerprint/{person_id} | ✅ | session | Active `{templates}` |
| GET /api/v1/biometric/fingerprint/{person_id}/{finger} | ✅ | session | Single finger |
| DELETE /api/v1/biometric/fingerprint/{person_id}/{finger} | ✅ | session | Deactivate → 204 |
| POST /api/v1/biometric/fingerprint/verify | ✅ | session | 1:1 |
| POST /api/v1/biometric/fingerprint/identify | ✅ | session | 1:N + finger filter |
| POST /api/v1/biometric/update-request | ✅ | session | Mirror → 201 Pending (thin proxy of canonical case in configuration) |
| GET /api/v1/biometric/update-requests/{person_id} | ✅ | session | By person |
| PATCH /api/v1/biometric/update-request/{request_id}/review | ✅ | session | Review |
| DELETE /api/v1/biometric/update-request/{request_id} | ✅ | session | Delete → 204 |

## 7. Configuration 07 (`07-ms-configuration`, TS 8087 → `07-configuration-service.yaml`)

| METHOD path | Kong | Auth | Description |
|-------------|------|------|-------------|
| POST/GET /api/v1/configurations/academic (+alias /academic-configurations) | ✅ | session | CRUD + `?name&limit&offset` |
| GET/PUT/DELETE /api/v1/configurations/academic/{id} (+alias) | ✅ | session | CRUD |
| POST/GET /api/v1/configurations/security (+alias /security-configurations) | ✅ | session | CRUD (no schoolId) |
| GET/PUT/DELETE /api/v1/configurations/security/{id} (+alias) | ✅ | session | CRUD |
| GET /api/v1/schools/{schoolId}/configurations | ✅ nested | session | By school |
| POST/GET /api/v1/biometric-update-cases?status&limit&offset | ✅ | session | Cases (canonical route; logical owner Biometric) |
| GET/PUT/DELETE /api/v1/biometric-update-cases/{id} | ✅ | session | CRUD |
| PATCH /api/v1/biometric-update-cases/{id}/review | ✅ | session | `In_Review\|Approved\|Rejected` + `reviewedAt` (`requestedBy` mandatory) |
| GET /api/v1/persons/{personId}/biometric-cases | ✅ nested | session | By person |

## 8. Notification 08 (`08-ms-notification`, Go 8088 → `08-notification-service.yaml`)

| METHOD path | Kong | Auth | Description |
|-------------|------|------|-------------|
| POST /api/v1/alerts | ✅ | session | Create → 201 `{alert_id}` |
| GET /api/v1/alerts?actorId&typeId&resolved\|status&limit&offset | ✅ | session | List `{alerts}` |
| GET /api/v1/alerts/{id} | ✅ | session | Get |
| PATCH /api/v1/alerts/{id}/resolve | ✅ | session | Resolve |
| DELETE /api/v1/alerts/{id} | ✅ | session | Delete → 204 |
| POST /api/v1/alert-types | ✅ | session | Create → 201 (`severity` INFO/WARNING/CRITICAL, `channel` DASHBOARD/EMAIL/PUSH) |
| GET /api/v1/alert-types?limit=50&offset=0 | ✅ | session | List `{alert_types}` |
| GET/PUT/DELETE /api/v1/alert-types/{id} | ✅ | session | CRUD |
| POST /api/v1/emails | ❌ no Kong route | internal only | Send a recovery email (`{to,template,data}`, template-only to avoid an open relay) → 202; optional `X-Internal-Token`; reachable only inside the compose network (called by 01-ms-identity) |

## 9. Quality 09 (`09-ms-quality`, TS 8089 → `09-quality-service.yaml`)

| METHOD path | Kong | Auth | Description |
|-------------|------|------|-------------|
| GET /api/v1/quality/characteristics | ✅ | session | 25010 instrument (24) |
| POST /api/v1/quality/evaluations | ✅ | session | Create (24 scores) → 201 |
| GET /api/v1/quality/evaluations?service&status&limit&offset | ✅ | session | List `{data,total,limit,offset}` |
| GET/PUT/DELETE /api/v1/quality/evaluations/{id} | ✅ | session | CRUD |
| GET /api/v1/quality/services/{service}/summary | ✅ | session | Aggregate |
| GET /api/v1/quality/process/profile | ✅ | session | 29110 instrument (20) |
| POST/GET /api/v1/quality/projects, GET/PUT/DELETE /{id} | ✅ | session | CRUD projects |
| POST /api/v1/quality/assessments | ✅ | session | Create (project + coverage) → 201 |
| GET /api/v1/quality/assessments?projectId&processId | ✅ | session | List |
| GET/PUT/DELETE /api/v1/quality/assessments/{id} | ✅ | session | CRUD |
| GET /api/v1/quality/projects/{id}/summary | ✅ | session | Summary + readiness |
| GET /api/v1/quality/istqb/categories (+alias characteristics) | ✅ | session | ISTQB instrument (30) |
| POST /api/v1/quality/istqb/assessments | ✅ | session | Create → 201 |
| GET /api/v1/quality/istqb/assessments?service&status&projectId | ✅ | session | List |
| GET/PUT/DELETE /api/v1/quality/istqb/assessments/{id} | ✅ | session | CRUD |
| GET /api/v1/quality/istqb/services/{service}/summary | ✅ | session | Aggregate |

## 10. Gateway 99 (Kong 8080 → `api-gateway.yaml`)

See table in `authentication.md §2` and `kong/kong.yml`. Global plugins: `pre-function` (edge
authentication), `cors` (origins `localhost:3000`, `5173`, `8090`) and `rate-limiting 300/min`
(redis policy), plus per-service redis limits (identity 60, authorization 100, attendance 200,
quality 100 per minute). No `jwt` plugin and no consumers.

### 10.1 Edge authentication (global `pre-function`, Lua, `kong.yml`)

| Item | Behaviour |
|------|-----------|
| Token | `Authorization: Bearer <uuid>` — the opaque `sessionId` from `POST /api/v1/auth/login`, **not** a JWT |
| Check | `GET http://ms-identity:8081/api/v1/sessions/{id}` (connect/send 2 s, read 3 s); accepted only on HTTP 200 with `"sessionStatus":"Active"` |
| 401 | No `Authorization` header, not `Bearer`, token not an 8-4-4-4-12 UUID, or session invalid/expired/closed → `{"status":401,"error":"Unauthorized","message":"Missing or invalid bearer token" \| "Invalid or expired session"}` |
| 503 | Identity unreachable/timeout → `{"status":503,"error":"Service Unavailable","message":"Cannot verify session"}` |
| Cache | **None** — one identity call per authenticated request |
| Public routes | `OPTIONS` (any path, CORS), `/health*`, `/api/v1/health`, `/api/health`, `POST /api/v1/auth/{login,logout,refresh,forgot-password,verify-code,reset-password}`, `GET /api/v1/auth/me` |
| Authorization | Not evaluated at the gateway — every service evaluates RBAC (§11) |
| Requirement | `KONG_UNTRUSTED_LUA_SANDBOX_REQUIRES=resty.http` on the gateway container (`back-end/docker-compose.yml`) |

`GET /health/quality` is the only health route at the gateway (proxied to
`ms-academic:8083`); there is **no** `GET /health` route — probe each service directly on its port.

## 11. Required permission per route (RBAC)

Every service validates the session against identity and evaluates the permission through
`GET /api/v1/auth/evaluate` (Java `AuthTokenFilter`, TS `authGuard.ts`, Python `security.py`,
Go `auth_middleware.go`). Seeded grants per role: `database/SEEDS.md` §3.

| Service | Rule | Permission |
|---------|------|------------|
| 01 identity | `GET /persons`, `GET /persons/{id}` | `identity.person:read` + scope (below) |
| 01 identity | `GET /users/{id}` | self, or `identity.person:write` |
| 01 identity | `GET /users`, `GET /sessions`, `GET /sessions/user/{other}` | `identity.person:write` |
| 01 identity | `PUT /persons/{ownPersonId}` (status and document are kept) | `identity.person:read` |
| 01 identity | `POST /auth/change-password` | session only |
| 01 identity | any other write | `identity.person:write` |
| 02 authorization | own `/users/{self}/roles`, `/auth/evaluate` | session only |
| 02 authorization | everything else | `configuration:manage` |
| 03 academic | `GET` | session only |
| 03 academic | enrollment writes / other writes | `academic.enrollment:manage` / `academic.school:manage` |
| 04 scheduling | `GET` | `attendance.record:read` |
| 04 scheduling | environment and schedule-block writes | `academic.school:manage` |
| 04 scheduling | class-session writes, `open/close/cancel` | `scheduling.session:manage` |
| 05 attendance | `GET` | `attendance.record:read` |
| 05 attendance | `POST /justifications`, `POST /supporting-documents` | `attendance.justification:request` |
| 05 attendance | `PATCH /justifications/{id}/review` | `attendance.justification:approve` |
| 05 attendance | justification-type writes | `configuration:manage` |
| 05 attendance | any other write | `attendance.record:write` |
| 06 biometric | `liveness-challenge`, `liveness-step`, `enroll-image` | `biometric.case:request` for the caller's own person; replacing a template needs a case `Approved` after it (RN-29). Staff (`attendance.record:write`) enrolls anyone |
| 06 biometric | other facial and fingerprint routes | `attendance.record:write` |
| 07 configuration | biometric cases `GET`/`POST` (own cases) · review, `PUT`, `DELETE` | `biometric.case:request` · `biometric.case:review` (never your own request) |
| 07 configuration | configuration writes | `configuration:manage` |
| 08 notification | `GET` · `PATCH /alerts/{id}/resolve` · other writes | `notification.alert:read` · `attendance.record:write` · `configuration:manage` |
| 09 quality | everything | `configuration:manage` |

**Data scope (non-admins).** On top of the permission, reads are narrowed to the caller:

| Role | Persons (01) | Attendance records, summary, report, justifications, documents (05) · alerts (08) |
|------|--------------|------------------|
| Administrador / Super Administrador | all | all |
| Instructor | self + active students of the cohorts they teach | own actors + those students |
| Aprendiz | self + instructors of their cohorts | own actors only |

Out-of-scope items answer 403; lists simply omit them. Writes are checked too (e.g. a
justification can only be filed for an attendance record in scope). The scope is resolved with
the caller's bearer against identity, academic (`academic-actors`, `enrollments`) and scheduling
(`schedule-blocks`), cached 60 s; if those do not answer the request fails closed with 503.
