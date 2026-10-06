# Endpoint Matrix — FaceAttendEDU

> Cross-service canonical table. `Kong` = path via `:8080`; `Direct` = `http://localhost:<port>`.
> `Auth`: `public` (no session required) · `session` (Bearer opaque `sessionId`, validated by the
> MS-side `AuthTokenFilter`) · `—` (no gateway or MS-side auth on that route today).
> Kong runs **no jwt plugin and no consumers** (only `cors` + `rate-limiting`); auth is enforced
> per-MS by the Java services (01/02/04/05), never at the edge. Details per contract in `contracts/openapi/`.

## 1. Identity 01 (`01-ms-identity`, Java 8081 → `01-identity-service.yaml`)

| METHOD path | Kong | Auth | Description |
|-------------|------|------|-------------|
| POST /api/v1/auth/login | ✅ | public | Login (`identifier` = email **or** username) → `{sessionId,userId,sessionStatus}` |
| POST /api/v1/auth/logout?sessionId | ✅ | public | Logout → 204 |
| GET /api/v1/auth/me?username\|email | ✅ | public | Current user → `UserDto` (params optional) |
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
| GET /api/v1/sessions/{id} | ✅ | public | Get by UUID (session lookup used by every `AuthTokenFilter`) |
| DELETE /api/v1/sessions/{id} | ✅ | session | Close → 204 |
| GET /api/v1/sessions/user/{userId} | ✅ | public | By user |
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
| GET /api/v1/auth/evaluate?userId&permission | ✅ | session | RBAC → `{allowed}` |

## 3. Academic 03 (`03-ms-academic`, TS 8083 → `03-academic-service.yaml`)

| METHOD path | Kong | Auth | Description |
|-------------|------|------|-------------|
| POST/GET /api/v1/schools, GET/PUT/DELETE /api/v1/schools/{id} | ✅ | — | CRUD + `?limit&offset` + `x-total-count` |
| PATCH /api/v1/schools/{id}/status | ✅ | — | Status |
| POST /api/v1/schools/{schoolId}/programs, GET …/programs | ✅ nested | — | By school |
| POST/GET /api/v1/programs, GET/PUT/DELETE /api/v1/programs/{id} | ✅ | — | CRUD |
| POST/GET /api/v1/academic-periods (+alias /periods), GET/PUT/DELETE /{id} | ✅ | — | CRUD per alias |
| POST/GET /api/v1/schools/{schoolId}/periods | ✅ nested | — | By school |
| POST/GET /api/v1/cohorts, GET/PUT/DELETE /api/v1/cohorts/{id} | ✅ | — | CRUD |
| GET /api/v1/programs/{programId}/cohorts | ✅ nested | — | By program |
| POST /api/v1/programs/{programId}/courses, POST/GET /api/v1/courses, GET/PUT/DELETE /{id} | ✅ | — | CRUD + by program |
| GET /api/v1/programs/{programId}/courses | ✅ nested | — | By program |
| POST/GET /api/v1/actor-types, GET/PUT/DELETE /{id} | ✅ | — | CRUD |
| POST/GET /api/v1/academic-actors, GET/PUT/DELETE /{id} | ✅ | — | CRUD |
| PATCH /api/v1/academic-actors/{id}/status | ✅ | — | Status |
| GET /api/v1/schools/{schoolId}/actors | ✅ nested | — | By school |
| POST/GET /api/v1/enrollments, GET/PUT/DELETE /{id} | ✅ | — | CRUD |
| PATCH /api/v1/enrollments/{id}/status | ✅ | — | `Active\|Withdrawn\|Completed` |
| GET /api/v1/cohorts/{cohortId}/enrollments | ✅ nested | — | By cohort |
| GET /api/v1/academic-actors/{actorId}/enrollments | ✅ nested | — | By actor |

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
| POST /api/v1/attendance-records/bulk | ✅ | session | Bulk → 201 |
| GET/PUT/DELETE /api/v1/attendance-records/{id} | ✅ | session | CRUD |
| GET /api/v1/justifications?status&attendanceRecordId | ✅ | session | List filtered |
| POST /api/v1/justifications | ✅ | session | Create → 201 |
| GET/DELETE /api/v1/justifications/{id} | ✅ | session | Read/Delete |
| PATCH /api/v1/justifications/{id}/review | ✅ | session | Review |
| GET /api/v1/justification-types | ✅ | session | List (`?schoolId` per-school, NULL = global) |
| POST /api/v1/justification-types | ✅ | session | Create → 201 |
| GET/PUT/DELETE /api/v1/justification-types/{id} | ✅ | session | CRUD |
| POST/GET /api/v1/attendance-reports | ❌ | session | Planned — `attendance_report` derived projection owned by 05-attendance, to implement |
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
| POST /api/v1/biometric/facial/enroll | ✅ | — | Enroll → 201 (0.85 threshold) |
| GET /api/v1/biometric/facial/{person_id} | ✅ | — | Active / 404 |
| GET /api/v1/biometric/facial/{person_id}/history | ✅ | — | Versions |
| DELETE /api/v1/biometric/facial/{person_id} | ✅ | — | Deactivate → 204 |
| POST /api/v1/biometric/facial/verify | ✅ | — | 1:1 `{match,score}` |
| POST /api/v1/biometric/facial/identify | ✅ | — | 1:N / 404 |
| POST /api/v1/biometric/fingerprint/enroll | ✅ | — | Enroll → 201 |
| GET /api/v1/biometric/fingerprint/{person_id} | ✅ | — | Active `{templates}` |
| GET /api/v1/biometric/fingerprint/{person_id}/{finger} | ✅ | — | Single finger |
| DELETE /api/v1/biometric/fingerprint/{person_id}/{finger} | ✅ | — | Deactivate → 204 |
| POST /api/v1/biometric/fingerprint/verify | ✅ | — | 1:1 |
| POST /api/v1/biometric/fingerprint/identify | ✅ | — | 1:N + finger filter |
| POST /api/v1/biometric/update-request | ✅ | — | Mirror → 201 Pending (thin proxy of canonical case in configuration) |
| GET /api/v1/biometric/update-requests/{person_id} | ✅ | — | By person |
| PATCH /api/v1/biometric/update-request/{request_id}/review | ✅ | — | Review |
| DELETE /api/v1/biometric/update-request/{request_id} | ✅ | — | Delete → 204 |

## 7. Configuration 07 (`07-ms-configuration`, TS 8087 → `07-configuration-service.yaml`)

| METHOD path | Kong | Auth | Description |
|-------------|------|------|-------------|
| POST/GET /api/v1/configurations/academic (+alias /academic-configurations) | ✅ | — | CRUD + `?name&limit&offset` |
| GET/PUT/DELETE /api/v1/configurations/academic/{id} (+alias) | ✅ | — | CRUD |
| POST/GET /api/v1/configurations/security (+alias /security-configurations) | ✅ | — | CRUD (no schoolId) |
| GET/PUT/DELETE /api/v1/configurations/security/{id} (+alias) | ✅ | — | CRUD |
| GET /api/v1/schools/{schoolId}/configurations | ✅ nested | — | By school |
| POST/GET /api/v1/biometric-update-cases?status&limit&offset | ✅ | — | Cases (canonical route; logical owner Biometric) |
| GET/PUT/DELETE /api/v1/biometric-update-cases/{id} | ✅ | — | CRUD |
| PATCH /api/v1/biometric-update-cases/{id}/review | ✅ | — | `In_Review\|Approved\|Rejected` + `reviewedAt` (`requestedBy` mandatory) |
| GET /api/v1/persons/{personId}/biometric-cases | ✅ nested | — | By person |

## 8. Notification 08 (`08-ms-notification`, Go 8088 → `08-notification-service.yaml`)

| METHOD path | Kong | Auth | Description |
|-------------|------|------|-------------|
| POST /api/v1/alerts | ✅ | — | Create → 201 `{alert_id}` |
| GET /api/v1/alerts?actorId&typeId&resolved\|status&limit&offset | ✅ | — | List `{alerts}` |
| GET /api/v1/alerts/{id} | ✅ | — | Get |
| PATCH /api/v1/alerts/{id}/resolve | ✅ | — | Resolve |
| DELETE /api/v1/alerts/{id} | ✅ | — | Delete → 204 |
| POST /api/v1/alert-types | ✅ | — | Create → 201 (`severity` INFO/WARNING/CRITICAL, `channel` DASHBOARD/EMAIL/PUSH) |
| GET /api/v1/alert-types?limit=50&offset=0 | ✅ | — | List `{alert_types}` |
| GET/PUT/DELETE /api/v1/alert-types/{id} | ✅ | — | CRUD |

## 9. Quality 09 (`09-ms-quality`, TS 8089 → `09-quality-service.yaml`)

| METHOD path | Kong | Auth | Description |
|-------------|------|------|-------------|
| GET /api/v1/quality/characteristics | ✅ | — | 25010 instrument (24) |
| POST /api/v1/quality/evaluations | ✅ | — | Create (24 scores) → 201 |
| GET /api/v1/quality/evaluations?service&status&limit&offset | ✅ | — | List `{data,total,limit,offset}` |
| GET/PUT/DELETE /api/v1/quality/evaluations/{id} | ✅ | — | CRUD |
| GET /api/v1/quality/services/{service}/summary | ✅ | — | Aggregate |
| GET /api/v1/quality/process/profile | ✅ | — | 29110 instrument (20) |
| POST/GET /api/v1/quality/projects, GET/PUT/DELETE /{id} | ✅ | — | CRUD projects |
| POST /api/v1/quality/assessments | ✅ | — | Create (project + coverage) → 201 |
| GET /api/v1/quality/assessments?projectId&processId | ✅ | — | List |
| GET/PUT/DELETE /api/v1/quality/assessments/{id} | ✅ | — | CRUD |
| GET /api/v1/quality/projects/{id}/summary | ✅ | — | Summary + readiness |
| GET /api/v1/quality/istqb/categories (+alias characteristics) | ✅ | — | ISTQB instrument (30) |
| POST /api/v1/quality/istqb/assessments | ✅ | — | Create → 201 |
| GET /api/v1/quality/istqb/assessments?service&status&projectId | ✅ | — | List |
| GET/PUT/DELETE /api/v1/quality/istqb/assessments/{id} | ✅ | — | CRUD |
| GET /api/v1/quality/istqb/services/{service}/summary | ✅ | — | Aggregate |

## 10. Gateway 99 (Kong 8080 → `api-gateway.yaml`)

See table in `authentication.md §2` and `kong/kong.yml`. Global `cors` (origins `localhost:3000`,
`5173`, `8090`) + `rate-limiting 300/min` (redis policy), plus per-service redis limits
(identity 60, authorization 100, attendance 200, quality 100 per minute). No `jwt` plugin and no
consumers. `GET /health/quality` is the only health route at the gateway (proxied to
`ms-academic:8083`); there is **no** `GET /health` route — probe each service directly on its port.
