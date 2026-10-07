# Endpoint Matrix — FaceAttendEDU

> Cross-service canonical table. `Kong` = path via `:8080`; `Direct` = `http://localhost:<port>`.
> `Auth`: `public` (no session required) · `session` (Bearer opaque `sessionId`) ·
> `gateway only` (only Kong checks the session — the MS itself has no auth filter) ·
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
> `jwt` plugin nor consumers. The gateway **only authenticates**: permissions (RBAC) are still
> evaluated by the Java services (01/02/04/05) with `AuthTokenFilter`. Full rules and public-route
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
| POST/GET /api/v1/schools, GET/PUT/DELETE /api/v1/schools/{id} | ✅ | gateway only | CRUD + `?limit&offset` + `x-total-count` |
| PATCH /api/v1/schools/{id}/status | ✅ | gateway only | Status |
| POST /api/v1/schools/{schoolId}/programs, GET …/programs | ✅ nested | gateway only | By school |
| POST/GET /api/v1/programs, GET/PUT/DELETE /api/v1/programs/{id} | ✅ | gateway only | CRUD |
| POST/GET /api/v1/academic-periods (+alias /periods), GET/PUT/DELETE /{id} | ✅ | gateway only | CRUD per alias |
| POST/GET /api/v1/schools/{schoolId}/periods | ✅ nested | gateway only | By school |
| POST/GET /api/v1/cohorts, GET/PUT/DELETE /api/v1/cohorts/{id} | ✅ | gateway only | CRUD |
| GET /api/v1/programs/{programId}/cohorts | ✅ nested | gateway only | By program |
| POST /api/v1/programs/{programId}/courses, POST/GET /api/v1/courses, GET/PUT/DELETE /{id} | ✅ | gateway only | CRUD + by program |
| GET /api/v1/programs/{programId}/courses | ✅ nested | gateway only | By program |
| POST/GET /api/v1/actor-types, GET/PUT/DELETE /{id} | ✅ | gateway only | CRUD |
| POST/GET /api/v1/academic-actors, GET/PUT/DELETE /{id} | ✅ | gateway only | CRUD |
| PATCH /api/v1/academic-actors/{id}/status | ✅ | gateway only | Status |
| GET /api/v1/schools/{schoolId}/actors | ✅ nested | gateway only | By school |
| POST/GET /api/v1/enrollments, GET/PUT/DELETE /{id} | ✅ | gateway only | CRUD |
| PATCH /api/v1/enrollments/{id}/status | ✅ | gateway only | `Active\|Withdrawn\|Completed` |
| GET /api/v1/cohorts/{cohortId}/enrollments | ✅ nested | gateway only | By cohort |
| GET /api/v1/academic-actors/{actorId}/enrollments | ✅ nested | gateway only | By actor |

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
| POST /api/v1/biometric/facial/enroll | ✅ | gateway only | Enroll → 201 (0.85 threshold) |
| GET /api/v1/biometric/facial/{person_id} | ✅ | gateway only | Active / 404 |
| GET /api/v1/biometric/facial/{person_id}/history | ✅ | gateway only | Versions |
| DELETE /api/v1/biometric/facial/{person_id} | ✅ | gateway only | Deactivate → 204 |
| POST /api/v1/biometric/facial/verify | ✅ | gateway only | 1:1 `{match,score}` |
| POST /api/v1/biometric/facial/identify | ✅ | gateway only | 1:N / 404 |
| POST /api/v1/biometric/fingerprint/enroll | ✅ | gateway only | Enroll → 201 |
| GET /api/v1/biometric/fingerprint/{person_id} | ✅ | gateway only | Active `{templates}` |
| GET /api/v1/biometric/fingerprint/{person_id}/{finger} | ✅ | gateway only | Single finger |
| DELETE /api/v1/biometric/fingerprint/{person_id}/{finger} | ✅ | gateway only | Deactivate → 204 |
| POST /api/v1/biometric/fingerprint/verify | ✅ | gateway only | 1:1 |
| POST /api/v1/biometric/fingerprint/identify | ✅ | gateway only | 1:N + finger filter |
| POST /api/v1/biometric/update-request | ✅ | gateway only | Mirror → 201 Pending (thin proxy of canonical case in configuration) |
| GET /api/v1/biometric/update-requests/{person_id} | ✅ | gateway only | By person |
| PATCH /api/v1/biometric/update-request/{request_id}/review | ✅ | gateway only | Review |
| DELETE /api/v1/biometric/update-request/{request_id} | ✅ | gateway only | Delete → 204 |

## 7. Configuration 07 (`07-ms-configuration`, TS 8087 → `07-configuration-service.yaml`)

| METHOD path | Kong | Auth | Description |
|-------------|------|------|-------------|
| POST/GET /api/v1/configurations/academic (+alias /academic-configurations) | ✅ | gateway only | CRUD + `?name&limit&offset` |
| GET/PUT/DELETE /api/v1/configurations/academic/{id} (+alias) | ✅ | gateway only | CRUD |
| POST/GET /api/v1/configurations/security (+alias /security-configurations) | ✅ | gateway only | CRUD (no schoolId) |
| GET/PUT/DELETE /api/v1/configurations/security/{id} (+alias) | ✅ | gateway only | CRUD |
| GET /api/v1/schools/{schoolId}/configurations | ✅ nested | gateway only | By school |
| POST/GET /api/v1/biometric-update-cases?status&limit&offset | ✅ | gateway only | Cases (canonical route; logical owner Biometric) |
| GET/PUT/DELETE /api/v1/biometric-update-cases/{id} | ✅ | gateway only | CRUD |
| PATCH /api/v1/biometric-update-cases/{id}/review | ✅ | gateway only | `In_Review\|Approved\|Rejected` + `reviewedAt` (`requestedBy` mandatory) |
| GET /api/v1/persons/{personId}/biometric-cases | ✅ nested | gateway only | By person |

## 8. Notification 08 (`08-ms-notification`, Go 8088 → `08-notification-service.yaml`)

| METHOD path | Kong | Auth | Description |
|-------------|------|------|-------------|
| POST /api/v1/alerts | ✅ | gateway only | Create → 201 `{alert_id}` |
| GET /api/v1/alerts?actorId&typeId&resolved\|status&limit&offset | ✅ | gateway only | List `{alerts}` |
| GET /api/v1/alerts/{id} | ✅ | gateway only | Get |
| PATCH /api/v1/alerts/{id}/resolve | ✅ | gateway only | Resolve |
| DELETE /api/v1/alerts/{id} | ✅ | gateway only | Delete → 204 |
| POST /api/v1/alert-types | ✅ | gateway only | Create → 201 (`severity` INFO/WARNING/CRITICAL, `channel` DASHBOARD/EMAIL/PUSH) |
| GET /api/v1/alert-types?limit=50&offset=0 | ✅ | gateway only | List `{alert_types}` |
| GET/PUT/DELETE /api/v1/alert-types/{id} | ✅ | gateway only | CRUD |
| POST /api/v1/emails | ❌ no Kong route | internal only | Send a recovery email (`{to,template,data}`, template-only to avoid an open relay) → 202; optional `X-Internal-Token`; reachable only inside the compose network (called by 01-ms-identity) |

## 9. Quality 09 (`09-ms-quality`, TS 8089 → `09-quality-service.yaml`)

| METHOD path | Kong | Auth | Description |
|-------------|------|------|-------------|
| GET /api/v1/quality/characteristics | ✅ | gateway only | 25010 instrument (24) |
| POST /api/v1/quality/evaluations | ✅ | gateway only | Create (24 scores) → 201 |
| GET /api/v1/quality/evaluations?service&status&limit&offset | ✅ | gateway only | List `{data,total,limit,offset}` |
| GET/PUT/DELETE /api/v1/quality/evaluations/{id} | ✅ | gateway only | CRUD |
| GET /api/v1/quality/services/{service}/summary | ✅ | gateway only | Aggregate |
| GET /api/v1/quality/process/profile | ✅ | gateway only | 29110 instrument (20) |
| POST/GET /api/v1/quality/projects, GET/PUT/DELETE /{id} | ✅ | gateway only | CRUD projects |
| POST /api/v1/quality/assessments | ✅ | gateway only | Create (project + coverage) → 201 |
| GET /api/v1/quality/assessments?projectId&processId | ✅ | gateway only | List |
| GET/PUT/DELETE /api/v1/quality/assessments/{id} | ✅ | gateway only | CRUD |
| GET /api/v1/quality/projects/{id}/summary | ✅ | gateway only | Summary + readiness |
| GET /api/v1/quality/istqb/categories (+alias characteristics) | ✅ | gateway only | ISTQB instrument (30) |
| POST /api/v1/quality/istqb/assessments | ✅ | gateway only | Create → 201 |
| GET /api/v1/quality/istqb/assessments?service&status&projectId | ✅ | gateway only | List |
| GET/PUT/DELETE /api/v1/quality/istqb/assessments/{id} | ✅ | gateway only | CRUD |
| GET /api/v1/quality/istqb/services/{service}/summary | ✅ | gateway only | Aggregate |

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
| Authorization | Not evaluated at the gateway — RBAC stays in the Java services (01/02/04/05) via `AuthTokenFilter` |
| Requirement | `KONG_UNTRUSTED_LUA_SANDBOX_REQUIRES=resty.http` on the gateway container (`back-end/docker-compose.yml`) |

`GET /health/quality` is the only health route at the gateway (proxied to
`ms-academic:8083`); there is **no** `GET /health` route — probe each service directly on its port.
