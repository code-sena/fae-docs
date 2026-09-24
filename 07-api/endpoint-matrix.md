# Endpoint Matrix — FaceAttendEDU

> Cross-service canonical table. `Kong` = path via `:8080`; `Direct` = `http://localhost:<port>`.
> `Auth`: `public` (identity, sin jwt en Kong) o `jwt` (resto). Detalle por contrato en `contracts/openapi/`.

## 1. Identity 01 (`01-ms-identity`, Java 8081 → `01-identity-service.yaml`)

| METHOD path | Kong | Auth | Descripción |
|-------------|------|------|-------------|
| POST /api/v1/auth/login | ✅ | public | Login → `{sessionId,userId,sessionStatus}` |
| POST /api/v1/auth/logout?sessionId | ✅ | public | Logout → 204 |
| GET /api/v1/auth/me?username | ✅ | public | Usuario actual |
| POST /api/v1/users | ✅ | public | Crear usuario |
| GET /api/v1/users/{id} | ✅ | public | Obtener usuario |
| PUT /api/v1/users/{id} | ✅ | public | Actualizar → 204 |
| POST /api/v1/users/{id}/activate | ✅ nested | public | Activar → 204 |
| PATCH /api/v1/users/{id}/status | ✅ | public | Estado → 204 |
| POST /api/v1/persons | ✅ | public | Crear persona |
| GET /api/v1/persons/{id} | ✅ | public | Obtener persona |
| PUT /api/v1/persons/{id} | ✅ | public | Actualizar → 204 |
| POST /api/v1/persons/{id}/transfer | ✅ nested | public | Transferir `{newSchoolId}` → 204 |
| PATCH /api/v1/persons/{id}/status | ✅ | public | Estado → 204 |
| POST /api/v1/cities | ✅ | public | Crear ciudad → 201 |
| GET /api/v1/cities?limit&offset | ✅ | public | Listar (paginado real) |
| GET /api/v1/cities/{id} | ✅ | public | Obtener ciudad |
| PUT /api/v1/cities/{id} | ✅ | public | Actualizar → 204 |
| DELETE /api/v1/cities/{id} | ✅ | public | Eliminar → 204 |
| POST /api/v1/password-policies | ✅ | public | Crear política → 201 |
| GET /api/v1/password-policies | ✅ | public | Listar |
| GET /api/v1/password-policies/{id} | ✅ | public | Obtener |
| PUT /api/v1/password-policies/{id} | ✅ | public | Actualizar → 204 |
| POST /api/v1/sessions?userId&sourceIp | ✅ | public | Crear sesión → 201 |
| DELETE /api/v1/sessions/{id} | ✅ | public | Cerrar → 204 |
| GET /api/v1/sessions/user/{userId} | ✅ | public | Por usuario |
| GET /health, /api/v1/health | direct | public | Health |

## 2. Authorization 02 (`02-ms-authorization`, Java 8083 → `02-authorization-service.yaml`)

| METHOD path | Kong | Auth | Descripción |
|-------------|------|------|-------------|
| GET /api/v1/roles | ✅ | jwt | Listar |
| POST /api/v1/roles | ✅ | jwt | Crear → 201 |
| GET/PUT/DELETE /api/v1/roles/{id} | ✅ | jwt | CRUD |
| GET /api/v1/roles/{roleId}/permissions | ✅ | jwt | Permisos del rol |
| POST /api/v1/roles/{roleId}/permissions | ✅ | jwt | Asignar → 201 |
| DELETE /api/v1/roles/{roleId}/permissions/{permId} | ✅ | jwt | Quitar → 204 |
| GET /api/v1/permissions | ✅ | jwt | Listar |
| POST /api/v1/permissions | ✅ | jwt | Crear → 201 |
| GET/PUT/DELETE /api/v1/permissions/{id} | ✅ | jwt | CRUD |
| POST /api/v1/users/{userId}/roles | ✅ nested | jwt | Asignar rol → 201 |
| DELETE /api/v1/users/{userId}/roles/{roleId} | ✅ | jwt | Quitar → 204 |
| GET /api/v1/users/{userId}/roles | ✅ | jwt | Roles del usuario |
| GET /api/v1/auth/evaluate?userId&permission | ✅ | jwt | RBAC → `{allowed}` |

## 3. Academic 03 (`03-ms-academic`, TS 8084 → `03-academic-service.yaml`)

| METHOD path | Kong | Auth | Descripción |
|-------------|------|------|-------------|
| POST/GET /api/v1/schools, GET/PUT/DELETE /api/v1/schools/{id} | ✅ | jwt | CRUD + `?limit&offset` + `x-total-count` |
| PATCH /api/v1/schools/{id}/status | ✅ | jwt | Estado |
| POST /api/v1/schools/{schoolId}/programs, GET …/programs | ✅ nested | jwt | Por escuela |
| POST/GET /api/v1/programs, GET/PUT/DELETE /api/v1/programs/{id} | ✅ | jwt | CRUD |
| POST/GET /api/v1/academic-periods (+alias /periods), GET/PUT/DELETE /{id} | ✅ | jwt | CRUD par alias |
| POST/GET /api/v1/schools/{schoolId}/periods | ✅ nested | jwt | Por escuela |
| POST/GET /api/v1/cohorts, GET/PUT/DELETE /api/v1/cohorts/{id} | ✅ | jwt | CRUD |
| GET /api/v1/programs/{programId}/cohorts | ✅ nested | jwt | Por programa |
| POST /api/v1/programs/{programId}/courses, POST/GET /api/v1/courses, GET/PUT/DELETE /{id} | ✅ | jwt | CRUD + por programa |
| GET /api/v1/programs/{programId}/courses | ✅ nested | jwt | Por programa |
| POST/GET /api/v1/actor-types, GET/PUT/DELETE /{id} | ✅ | jwt | CRUD |
| POST/GET /api/v1/academic-actors, GET/PUT/DELETE /{id} | ✅ | jwt | CRUD |
| PATCH /api/v1/academic-actors/{id}/status | ✅ | jwt | Estado |
| GET /api/v1/schools/{schoolId}/actors | ✅ nested | jwt | Por escuela |
| POST/GET /api/v1/enrollments, GET/PUT/DELETE /{id} | ✅ | jwt | CRUD |
| PATCH /api/v1/enrollments/{id}/status | ✅ | jwt | `Active\|Withdrawn\|Completed` |
| GET /api/v1/cohorts/{cohortId}/enrollments | ✅ nested | jwt | Por cohorte |
| GET /api/v1/academic-actors/{actorId}/enrollments | ✅ nested | jwt | Por actor |

## 4. Scheduling 04 (`04-ms-scheduling`, Java 8087 → `04-scheduling-service.yaml`)

| METHOD path | Kong | Auth | Descripción |
|-------------|------|------|-------------|
| GET /api/v1/environments?schoolId | ✅ | jwt | Listar filtrado |
| POST /api/v1/environments | ✅ | jwt | Crear → 201 |
| GET/PUT/DELETE /api/v1/environments/{id} | ✅ | jwt | CRUD |
| GET /api/v1/schedule-blocks?cohortId&environmentId&instructorActorId&courseId | ✅ | jwt | Listar filtrado |
| POST /api/v1/schedule-blocks | ✅ | jwt | Crear → 201 (409 si double-booking) |
| GET/PUT/DELETE /api/v1/schedule-blocks/{id} | ✅ | jwt | CRUD |
| GET /api/v1/class-sessions?scheduleBlockId | ✅ | jwt | Listar filtrado |
| POST /api/v1/class-sessions | ✅ | jwt | Crear → 201 |
| GET/PUT/DELETE /api/v1/class-sessions/{id} | ✅ | jwt | CRUD |
| POST /api/v1/class-sessions/{id}/open\|close\|cancel | ✅ | jwt | Transiciones (POST real) |
| GET /api/v1/cohorts/{id}/blocks | ✅ nested | jwt | Por cohorte |
| GET /api/v1/environments/{id}/blocks | ✅ | jwt | Por ambiente |
| GET /api/v1/actors/{id}/blocks | ✅ | jwt | Por instructor |
| GET /api/v1/blocks/{id}/sessions | ✅ | jwt | Sesiones del bloque |

## 5. Attendance 05 (`05-ms-attendance`, Java 8085 → `05-attendance-service.yaml`)

| METHOD path | Kong | Auth | Descripción |
|-------------|------|------|-------------|
| GET /api/v1/attendance-records?classSessionId&academicActorId&attendanceStatus | ✅ | jwt | Listar filtrado |
| POST /api/v1/attendance-records | ✅ | jwt | Crear → 201 |
| POST /api/v1/attendance-records/bulk | ✅ | jwt | Masivo → 201 |
| GET/PUT/DELETE /api/v1/attendance-records/{id} | ✅ | jwt | CRUD |
| GET /api/v1/justifications?status&attendanceRecordId | ✅ | jwt | Listar filtrado |
| POST /api/v1/justifications | ✅ | jwt | Crear → 201 |
| GET/DELETE /api/v1/justifications/{id} | ✅ | jwt | Read/Delete |
| PATCH /api/v1/justifications/{id}/review | ✅ | jwt | Revisar |
| GET /api/v1/justification-types | ✅ | jwt | Listar |
| POST /api/v1/justification-types | ✅ | jwt | Crear → 201 |
| GET/PUT/DELETE /api/v1/justification-types/{id} | ✅ | jwt | CRUD |
| GET /api/v1/supporting-documents?justificationId | ✅ | jwt | Listar filtrado |
| POST /api/v1/supporting-documents | ✅ | jwt | Crear → 201 |
| GET/DELETE /api/v1/supporting-documents/{id} | ✅ | jwt | Read/Delete |
| GET /api/v1/class-sessions/{id}/attendance | ✅ nested | jwt | Por sesión |
| GET /api/v1/academic-actors/{actorId}/attendance | ✅ nested | jwt | Por actor |
| GET /api/v1/attendance/{id}/justification | ✅ nested | jwt | Single\|null |
| GET /api/v1/justifications/{id}/documents | ✅ nested | jwt | Por justificación |

## 6. Biometric 06 (`06-ms-biometric`, Python 8086 → `06-biometric-service.yaml`)

| METHOD path | Kong | Auth | Descripción |
|-------------|------|------|-------------|
| POST /api/v1/biometric/facial/enroll | ✅ | jwt | Enrolar → 201 (0.85 threshold) |
| GET /api/v1/biometric/facial/{person_id} | ✅ | jwt | Activo / 404 |
| GET /api/v1/biometric/facial/{person_id}/history | ✅ | jwt | Versiones |
| DELETE /api/v1/biometric/facial/{person_id} | ✅ | jwt | Desactivar → 204 |
| POST /api/v1/biometric/facial/verify | ✅ | jwt | 1:1 `{match,score}` |
| POST /api/v1/biometric/facial/identify | ✅ | jwt | 1:N / 404 |
| POST /api/v1/biometric/fingerprint/enroll | ✅ | jwt | Enrolar → 201 |
| GET /api/v1/biometric/fingerprint/{person_id} | ✅ | jwt | Activos `{templates}` |
| GET /api/v1/biometric/fingerprint/{person_id}/{finger} | ✅ | jwt | Un dedo |
| DELETE /api/v1/biometric/fingerprint/{person_id}/{finger} | ✅ | jwt | Desactivar → 204 |
| POST /api/v1/biometric/fingerprint/verify | ✅ | jwt | 1:1 |
| POST /api/v1/biometric/fingerprint/identify | ✅ | jwt | 1:N + filtro dedo |
| POST /api/v1/biometric/update-request | ✅ | jwt | Espejo → 201 Pending |
| GET /api/v1/biometric/update-requests/{person_id} | ✅ | jwt | Por persona |
| PATCH /api/v1/biometric/update-request/{request_id}/review | ✅ | jwt | Revisar |
| DELETE /api/v1/biometric/update-request/{request_id} | ✅ | jwt | Eliminar → 204 |

## 7. Configuration 07 (`07-ms-configuration`, TS 8089 → `07-configuration-service.yaml`)

| METHOD path | Kong | Auth | Descripción |
|-------------|------|------|-------------|
| POST/GET /api/v1/configurations/academic (+alias /academic-configurations) | ✅ | jwt | CRUD + `?name&limit&offset` |
| GET/PUT/DELETE /api/v1/configurations/academic/{id} (+alias) | ✅ | jwt | CRUD |
| POST/GET /api/v1/configurations/security (+alias /security-configurations) | ✅ | jwt | CRUD (sin schoolId) |
| GET/PUT/DELETE /api/v1/configurations/security/{id} (+alias) | ✅ | jwt | CRUD |
| GET /api/v1/schools/{schoolId}/configurations | ✅ nested | jwt | Por escuela |
| POST/GET /api/v1/biometric-update-cases?status&limit&offset | ✅ | jwt | Casos (canónico) |
| GET/PUT/DELETE /api/v1/biometric-update-cases/{id} | ✅ | jwt | CRUD |
| PATCH /api/v1/biometric-update-cases/{id}/review | ✅ | jwt | `In_Review\|Approved\|Rejected` + `reviewedAt` |
| GET /api/v1/persons/{personId}/biometric-cases | ✅ nested | jwt | Por persona |

## 8. Notification 08 (`08-ms-notification`, Go 8090 → `08-notification-service.yaml`)

| METHOD path | Kong | Auth | Descripción |
|-------------|------|------|-------------|
| POST /api/v1/alerts | ✅ | jwt | Crear → 201 `{alert_id}` |
| GET /api/v1/alerts?actorId&typeId&resolved\|status&limit&offset | ✅ | jwt | Listar `{alerts}` |
| GET /api/v1/alerts/{id} | ✅ | jwt | Obtener |
| PATCH /api/v1/alerts/{id}/resolve | ✅ | jwt | Resolver |
| DELETE /api/v1/alerts/{id} | ✅ | jwt | Eliminar → 204 |
| POST /api/v1/alert-types | ✅ | jwt | Crear → 201 |
| GET /api/v1/alert-types?limit=50&offset=0 | ✅ | jwt | Listar `{alert_types}` |
| GET/PUT/DELETE /api/v1/alert-types/{id} | ✅ | jwt | CRUD |

## 9. Quality 09 (`09-ms-quality`, TS 8091 → `09-quality-service.yaml`)

| METHOD path | Kong | Auth | Descripción |
|-------------|------|------|-------------|
| GET /api/v1/quality/characteristics | ✅ | jwt | Instrumento 25010 (24) |
| POST /api/v1/quality/evaluations | ✅ | jwt | Crear (24 scores) → 201 |
| GET /api/v1/quality/evaluations?service&status&limit&offset | ✅ | jwt | Listar `{data,total,limit,offset}` |
| GET/PUT/DELETE /api/v1/quality/evaluations/{id} | ✅ | jwt | CRUD |
| GET /api/v1/quality/services/{service}/summary | ✅ | jwt | Agregado |
| GET /api/v1/quality/process/profile | ✅ | jwt | Instrumento 29110 (20) |
| POST/GET /api/v1/quality/projects, GET/PUT/DELETE /{id} | ✅ | jwt | CRUD proyectos |
| POST /api/v1/quality/assessments | ✅ | jwt | Crear (proyecto + cobertura) → 201 |
| GET /api/v1/quality/assessments?projectId&processId | ✅ | jwt | Listar |
| GET/PUT/DELETE /api/v1/quality/assessments/{id} | ✅ | jwt | CRUD |
| GET /api/v1/quality/projects/{id}/summary | ✅ | jwt | Resumen + readiness |
| GET /api/v1/quality/istqb/categories (+alias characteristics) | ✅ | jwt | Instrumento ISTQB (30) |
| POST /api/v1/quality/istqb/assessments | ✅ | jwt | Crear → 201 |
| GET /api/v1/quality/istqb/assessments?service&status&projectId | ✅ | jwt | Listar |
| GET/PUT/DELETE /api/v1/quality/istqb/assessments/{id} | ✅ | jwt | CRUD |
| GET /api/v1/quality/istqb/services/{service}/summary | ✅ | jwt | Agregado |

## 10. Gateway 99 (Kong 8080 → `api-gateway.yaml`)

Ver tabla en `authentication.md §2` y `kong/kong.yml`. Global `cors` + `rate-limiting 300/min`; `GET /health`, `GET /health/quality` públicos.
