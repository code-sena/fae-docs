# Endpoint Matrix — FaceAttendEDU

> Cross-service canonical table. `Kong` = path via `:8080`; `Direct` = `http://localhost:<port>`.
> `Auth`: `public` (identity, no jwt at Kong) or `jwt` (rest). Details per contract in `contracts/openapi/`.

## 1. Identity 01 (`01-ms-identity`, Java 8081 → `01-identity-service.yaml`)

| METHOD path | Kong | Auth | Description |
|-------------|------|------|-------------|
| POST /api/v1/auth/login | ✅ | public | Login → `{sessionId,userId,sessionStatus}` |
| POST /api/v1/auth/logout?sessionId | ✅ | public | Logout → 204 |
| GET /api/v1/auth/me?username | ✅ | public | Current user |
| POST /api/v1/users | ✅ | public | Create user |
| GET /api/v1/users/{id} | ✅ | public | Get user |
| PUT /api/v1/users/{id} | ✅ | public | Update → 204 |
| POST /api/v1/users/{id}/activate | ✅ nested | public | Activate → 204 |
| PATCH /api/v1/users/{id}/status | ✅ | public | Status → 204 |
| POST /api/v1/persons | ✅ | public | Create person |
| GET /api/v1/persons/{id} | ✅ | public | Get person |
| PUT /api/v1/persons/{id} | ✅ | public | Update → 204 |
| POST /api/v1/persons/{id}/transfer | ✅ nested | public | Transfer `{newSchoolId}` → 204 |
| PATCH /api/v1/persons/{id}/status | ✅ | public | Status → 204 |
| POST /api/v1/cities | ❌ deprecated | public | Removed — cities come from external `countriesnow.space` API |
| GET /api/v1/cities?limit&offset | ❌ deprecated | public | Removed — see external API |
| GET /api/v1/cities/{id} | ❌ deprecated | public | Removed — see external API |
| PUT /api/v1/cities/{id} | ❌ deprecated | public | Removed — see external API |
| DELETE /api/v1/cities/{id} | ❌ deprecated | public | Removed — see external API |
| POST /api/v1/password-policies | ✅ | public | Create policy → 201 |
| GET /api/v1/password-policies | ✅ | public | List |
| GET /api/v1/password-policies/{id} | ✅ | public | Get |
| PUT /api/v1/password-policies/{id} | ✅ | public | Update → 204 |
| POST /api/v1/sessions?userId&sourceIp | ✅ | public | Create session → 201 |
| DELETE /api/v1/sessions/{id} | ✅ | public | Close → 204 |
| GET /api/v1/sessions/user/{userId} | ✅ | public | By user |
| GET /health, /api/v1/health | direct | public | Health |

## 2. Authorization 02 (`02-ms-authorization`, Java 8083 → `02-authorization-service.yaml`)

| METHOD path | Kong | Auth | Description |
|-------------|------|------|-------------|
| GET /api/v1/roles | ✅ | jwt | List |
| POST /api/v1/roles | ✅ | jwt | Create → 201 |
| GET/PUT/DELETE /api/v1/roles/{id} | ✅ | jwt | CRUD |
| GET /api/v1/roles/{roleId}/permissions | ✅ | jwt | Role permissions |
| POST /api/v1/roles/{roleId}/permissions | ✅ | jwt | Assign → 201 |
| DELETE /api/v1/roles/{roleId}/permissions/{permId} | ✅ | jwt | Remove → 204 |
| GET /api/v1/permissions | ✅ | jwt | List |
| POST /api/v1/permissions | ✅ | jwt | Create → 201 |
| GET/PUT/DELETE /api/v1/permissions/{id} | ✅ | jwt | CRUD |
| POST /api/v1/users/{userId}/roles | ✅ nested | jwt | Assign role → 201 |
| DELETE /api/v1/users/{userId}/roles/{roleId} | ✅ | jwt | Remove → 204 |
| GET /api/v1/users/{userId}/roles | ✅ | jwt | User roles |
| GET /api/v1/auth/evaluate?userId&permission | ✅ | jwt | RBAC → `{allowed}` |

## 3. Academic 03 (`03-ms-academic`, TS 8084 → `03-academic-service.yaml`)

| METHOD path | Kong | Auth | Description |
|-------------|------|------|-------------|
| POST/GET /api/v1/schools, GET/PUT/DELETE /api/v1/schools/{id} | ✅ | jwt | CRUD + `?limit&offset` + `x-total-count` |
| PATCH /api/v1/schools/{id}/status | ✅ | jwt | Status |
| POST /api/v1/schools/{schoolId}/programs, GET …/programs | ✅ nested | jwt | By school |
| POST/GET /api/v1/programs, GET/PUT/DELETE /api/v1/programs/{id} | ✅ | jwt | CRUD |
| POST/GET /api/v1/academic-periods (+alias /periods), GET/PUT/DELETE /{id} | ✅ | jwt | CRUD per alias |
| POST/GET /api/v1/schools/{schoolId}/periods | ✅ nested | jwt | By school |
| POST/GET /api/v1/cohorts, GET/PUT/DELETE /api/v1/cohorts/{id} | ✅ | jwt | CRUD |
| GET /api/v1/programs/{programId}/cohorts | ✅ nested | jwt | By program |
| POST /api/v1/programs/{programId}/courses, POST/GET /api/v1/courses, GET/PUT/DELETE /{id} | ✅ | jwt | CRUD + by program |
| GET /api/v1/programs/{programId}/courses | ✅ nested | jwt | By program |
| POST/GET /api/v1/actor-types, GET/PUT/DELETE /{id} | ✅ | jwt | CRUD |
| POST/GET /api/v1/academic-actors, GET/PUT/DELETE /{id} | ✅ | jwt | CRUD |
| PATCH /api/v1/academic-actors/{id}/status | ✅ | jwt | Status |
| GET /api/v1/schools/{schoolId}/actors | ✅ nested | jwt | By school |
| POST/GET /api/v1/enrollments, GET/PUT/DELETE /{id} | ✅ | jwt | CRUD |
| PATCH /api/v1/enrollments/{id}/status | ✅ | jwt | `Active\|Withdrawn\|Completed` |
| GET /api/v1/cohorts/{cohortId}/enrollments | ✅ nested | jwt | By cohort |
| GET /api/v1/academic-actors/{actorId}/enrollments | ✅ nested | jwt | By actor |

## 4. Scheduling 04 (`04-ms-scheduling`, Java 8087 → `04-scheduling-service.yaml`)

| METHOD path | Kong | Auth | Description |
|-------------|------|------|-------------|
| GET /api/v1/environments?schoolId | ✅ | jwt | List filtered |
| POST /api/v1/environments | ✅ | jwt | Create → 201 |
| GET/PUT/DELETE /api/v1/environments/{id} | ✅ | jwt | CRUD |
| GET /api/v1/schedule-blocks?cohortId&environmentId&instructorActorId&courseId | ✅ | jwt | List filtered |
| POST /api/v1/schedule-blocks | ✅ | jwt | Create → 201 (409 on double-booking) |
| GET/PUT/DELETE /api/v1/schedule-blocks/{id} | ✅ | jwt | CRUD |
| GET /api/v1/class-sessions?scheduleBlockId | ✅ | jwt | List filtered |
| POST /api/v1/class-sessions | ✅ | jwt | Create → 201 |
| GET/PUT/DELETE /api/v1/class-sessions/{id} | ✅ | jwt | CRUD |
| POST /api/v1/class-sessions/{id}/open\|close\|cancel | ✅ | jwt | Transitions (real POST) |
| GET /api/v1/cohorts/{id}/blocks | ✅ nested | jwt | By cohort |
| GET /api/v1/environments/{id}/blocks | ✅ | jwt | By environment |
| GET /api/v1/actors/{id}/blocks | ✅ | jwt | By instructor |
| GET /api/v1/blocks/{id}/sessions | ✅ | jwt | Block sessions |

## 5. Attendance 05 (`05-ms-attendance`, Java 8085 → `05-attendance-service.yaml`)

| METHOD path | Kong | Auth | Description |
|-------------|------|------|-------------|
| GET /api/v1/attendance-records?classSessionId&academicActorId&attendanceStatus | ✅ | jwt | List filtered |
| POST /api/v1/attendance-records | ✅ | jwt | Create → 201 |
| POST /api/v1/attendance-records/bulk | ✅ | jwt | Bulk → 201 |
| GET/PUT/DELETE /api/v1/attendance-records/{id} | ✅ | jwt | CRUD |
| GET /api/v1/justifications?status&attendanceRecordId | ✅ | jwt | List filtered |
| POST /api/v1/justifications | ✅ | jwt | Create → 201 |
| GET/DELETE /api/v1/justifications/{id} | ✅ | jwt | Read/Delete |
| PATCH /api/v1/justifications/{id}/review | ✅ | jwt | Review |
| GET /api/v1/justification-types | ✅ | jwt | List (`?schoolId` per-school, NULL = global) |
| POST /api/v1/justification-types | ✅ | jwt | Create → 201 |
| GET/PUT/DELETE /api/v1/justification-types/{id} | ✅ | jwt | CRUD |
| POST/GET /api/v1/attendance-reports | ❌ | jwt | Missing — `attendance_report` derived projection, gap |
| GET /api/v1/supporting-documents?justificationId | ✅ | jwt | List filtered |
| POST /api/v1/supporting-documents | ✅ | jwt | Create → 201 |
| GET/DELETE /api/v1/supporting-documents/{id} | ✅ | jwt | Read/Delete |
| GET /api/v1/class-sessions/{id}/attendance | ✅ nested | jwt | By session |
| GET /api/v1/academic-actors/{actorId}/attendance | ✅ nested | jwt | By actor |
| GET /api/v1/attendance/{id}/justification | ✅ nested | jwt | Single\|null |
| GET /api/v1/justifications/{id}/documents | ✅ nested | jwt | By justification |

## 6. Biometric 06 (`06-ms-biometric`, Python 8086 → `06-biometric-service.yaml`)

| METHOD path | Kong | Auth | Description |
|-------------|------|------|-------------|
| POST /api/v1/biometric/facial/enroll | ✅ | jwt | Enroll → 201 (0.85 threshold) |
| GET /api/v1/biometric/facial/{person_id} | ✅ | jwt | Active / 404 |
| GET /api/v1/biometric/facial/{person_id}/history | ✅ | jwt | Versions |
| DELETE /api/v1/biometric/facial/{person_id} | ✅ | jwt | Deactivate → 204 |
| POST /api/v1/biometric/facial/verify | ✅ | jwt | 1:1 `{match,score}` |
| POST /api/v1/biometric/facial/identify | ✅ | jwt | 1:N / 404 |
| POST /api/v1/biometric/fingerprint/enroll | ✅ | jwt | Enroll → 201 |
| GET /api/v1/biometric/fingerprint/{person_id} | ✅ | jwt | Active `{templates}` |
| GET /api/v1/biometric/fingerprint/{person_id}/{finger} | ✅ | jwt | Single finger |
| DELETE /api/v1/biometric/fingerprint/{person_id}/{finger} | ✅ | jwt | Deactivate → 204 |
| POST /api/v1/biometric/fingerprint/verify | ✅ | jwt | 1:1 |
| POST /api/v1/biometric/fingerprint/identify | ✅ | jwt | 1:N + finger filter |
| POST /api/v1/biometric/update-request | ✅ | jwt | Mirror → 201 Pending (mirror of canonical case) |
| GET /api/v1/biometric/update-requests/{person_id} | ✅ | jwt | By person |
| PATCH /api/v1/biometric/update-request/{request_id}/review | ✅ | jwt | Review |
| DELETE /api/v1/biometric/update-request/{request_id} | ✅ | jwt | Delete → 204 |

## 7. Configuration 07 (`07-ms-configuration`, TS 8089 → `07-configuration-service.yaml`)

| METHOD path | Kong | Auth | Description |
|-------------|------|------|-------------|
| POST/GET /api/v1/configurations/academic (+alias /academic-configurations) | ✅ | jwt | CRUD + `?name&limit&offset` |
| GET/PUT/DELETE /api/v1/configurations/academic/{id} (+alias) | ✅ | jwt | CRUD |
| POST/GET /api/v1/configurations/security (+alias /security-configurations) | ✅ | jwt | CRUD (no schoolId) |
| GET/PUT/DELETE /api/v1/configurations/security/{id} (+alias) | ✅ | jwt | CRUD |
| GET /api/v1/schools/{schoolId}/configurations | ✅ nested | jwt | By school |
| POST/GET /api/v1/biometric-update-cases?status&limit&offset | ✅ | jwt | Cases (canonical route; logical owner Biometric) |
| GET/PUT/DELETE /api/v1/biometric-update-cases/{id} | ✅ | jwt | CRUD |
| PATCH /api/v1/biometric-update-cases/{id}/review | ✅ | jwt | `In_Review\|Approved\|Rejected` + `reviewedAt` (`requestedBy` mandatory) |
| GET /api/v1/persons/{personId}/biometric-cases | ✅ nested | jwt | By person |

## 8. Notification 08 (`08-ms-notification`, Go 8090 → `08-notification-service.yaml`)

| METHOD path | Kong | Auth | Description |
|-------------|------|------|-------------|
| POST /api/v1/alerts | ✅ | jwt | Create → 201 `{alert_id}` |
| GET /api/v1/alerts?actorId&typeId&resolved\|status&limit&offset | ✅ | jwt | List `{alerts}` |
| GET /api/v1/alerts/{id} | ✅ | jwt | Get |
| PATCH /api/v1/alerts/{id}/resolve | ✅ | jwt | Resolve |
| DELETE /api/v1/alerts/{id} | ✅ | jwt | Delete → 204 |
| POST /api/v1/alert-types | ✅ | jwt | Create → 201 (`severity` INFO/WARNING/CRITICAL, `channel` DASHBOARD/EMAIL/PUSH) |
| GET /api/v1/alert-types?limit=50&offset=0 | ✅ | jwt | List `{alert_types}` |
| GET/PUT/DELETE /api/v1/alert-types/{id} | ✅ | jwt | CRUD |

## 9. Quality 09 (`09-ms-quality`, TS 8091 → `09-quality-service.yaml`)

| METHOD path | Kong | Auth | Description |
|-------------|------|------|-------------|
| GET /api/v1/quality/characteristics | ✅ | jwt | 25010 instrument (24) |
| POST /api/v1/quality/evaluations | ✅ | jwt | Create (24 scores) → 201 |
| GET /api/v1/quality/evaluations?service&status&limit&offset | ✅ | jwt | List `{data,total,limit,offset}` |
| GET/PUT/DELETE /api/v1/quality/evaluations/{id} | ✅ | jwt | CRUD |
| GET /api/v1/quality/services/{service}/summary | ✅ | jwt | Aggregate |
| GET /api/v1/quality/process/profile | ✅ | jwt | 29110 instrument (20) |
| POST/GET /api/v1/quality/projects, GET/PUT/DELETE /{id} | ✅ | jwt | CRUD projects |
| POST /api/v1/quality/assessments | ✅ | jwt | Create (project + coverage) → 201 |
| GET /api/v1/quality/assessments?projectId&processId | ✅ | jwt | List |
| GET/PUT/DELETE /api/v1/quality/assessments/{id} | ✅ | jwt | CRUD |
| GET /api/v1/quality/projects/{id}/summary | ✅ | jwt | Summary + readiness |
| GET /api/v1/quality/istqb/categories (+alias characteristics) | ✅ | jwt | ISTQB instrument (30) |
| POST /api/v1/quality/istqb/assessments | ✅ | jwt | Create → 201 |
| GET /api/v1/quality/istqb/assessments?service&status&projectId | ✅ | jwt | List |
| GET/PUT/DELETE /api/v1/quality/istqb/assessments/{id} | ✅ | jwt | CRUD |
| GET /api/v1/quality/istqb/services/{service}/summary | ✅ | jwt | Aggregate |

## 10. Gateway 99 (Kong 8080 → `api-gateway.yaml`)

See table in `authentication.md §2` and `kong/kong.yml`. Global `cors` + `rate-limiting 300/min`; `GET /health`, `GET /health/quality` public.
