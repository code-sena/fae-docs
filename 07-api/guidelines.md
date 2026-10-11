# API Guidelines — FaceAttendEDU

> Canonical REST standards for all FaceAttendEDU microservices.
> Grounded in real code: `back-end/*/src`, `99-api-gateway/kong/kong.yml`, `front-end/Mobile|Web`.
> All new endpoints MUST follow this document before adding an OpenAPI contract.

## 1. Service map (real ports and stacks)

| # | Service | Code | Port | Stack (real) | DB schema |
|---|---------|------|------|--------------|-----------|
| 01 | Identity | `01-ms-identity` | 8081 | Java 21 + Spring Boot 4.1 | `identity` (Postgres) |
| 02 | Authorization | `02-ms-authorization` | 8082 | Java 21 + Spring Boot 4.1 | `authorization` (Postgres) |
| 03 | Academic | `03-ms-academic` | 8083 | TS Fastify + Zod + Drizzle | `academic` (Postgres) |
| 04 | Scheduling | `04-ms-scheduling` | 8084 | Java 21 + Spring Boot 4.1 | `scheduling` (Postgres) |
| 05 | Attendance | `05-ms-attendance` | 8085 | Java 21 + Spring Boot 4.1 | `attendance` (Postgres) |
| 06 | Biometric | `06-ms-biometric` | 8086 | Python 3.12 + FastAPI + Motor | MongoDB `facial_embedding`, `fingerprint_embedding` |
| 07 | Configuration | `07-ms-configuration` | 8087 | TS Fastify + Zod | `configuration` (Postgres) |
| 08 | Notification | `08-ms-notification` | 8088 | Go 1.22 + Gin + pgx | `notification` (Postgres) |
| 09 | Quality | `09-ms-quality` | 8089 | TS Fastify + Zod | in-memory + best-effort Postgres |
| 99 | Gateway | `99-api-gateway` | 8080 (proxy) / 8001 (admin) | Kong OSS 3.6 DB-less + Redis 7 | — |

Source of truth: `COMPOSE.md`, `back-end/SERVICES.md`, `back-end/docker-compose.yml`,
`99-api-gateway/kong/kong.yml`, root `.env.example`.

> Decisions 2026-10-03: `FULL/back-end/99-api-gateway/kong/kong.yml` is canonical; `contracts/openapi/api-gateway.yaml` is the published view. Probes are `/health` + `/health/ready` + `/health/quality` where implemented (the gateway itself only routes `/health/quality`).

## 2. Versioning

- **URL versioning only:** all canonical paths start with `/api/v1/`.
  Example: `GET http://localhost:8080/api/v1/persons` (via Kong) → `http://ms-identity:8081/api/v1/persons`.
- Direct (no-gateway) aliases without prefix exist for local dev (`/roles`, `/permissions`, `/users`, `/auth`, `/alerts`, …) but MUST NOT be documented as canonical and MUST NOT be used by frontends in staging/prod.
- Breaking change → new major `/api/v2/` alongside `/api/v1/` for ≥1 release. Never rename/remove a v1 field without deprecation header + `endpoint-matrix.md` update.
- Non-breaking (additive optional field, new filter, new nested read) → minor, same version.
- See `versioning.md` for deprecation policy.

## 3. Endpoint naming

- **Plural nouns, kebab-case:** `/api/v1/academic-actors`, `/api/v1/schedule-blocks`, `/api/v1/class-sessions`, `/api/v1/attendance-records`, `/api/v1/justification-types`, `/api/v1/supporting-documents`, `/api/v1/biometric-update-cases`, `/api/v1/alert-types`, `/api/v1/password-policies`.
- **No verbs in paths.** State transitions use sub-resources with POST/PATCH:
  - `POST /api/v1/class-sessions/{id}/open|close|cancel` (scheduling, real)
  - `PATCH /api/v1/justifications/{id}/review` (attendance, real)
  - `PATCH /api/v1/academic-actors/{id}/status`, `PATCH /api/v1/enrollments/{id}/status` (academic, real)
  - `PATCH /api/v1/alerts/{id}/resolve` (notification, real)
  - `PATCH /api/v1/biometric-update-cases/{id}/review` (canonical route in configuration; logical owner Biometric)
  - `POST /api/v1/users/{id}/activate`, `PATCH /api/v1/users/{id}/status` (identity, real); `POST /api/v1/persons/{id}/transfer` is **not implemented** (person has no `school_id`)
- **Nested reads** for ownership (mirrors `kong.yml` nested routes):
  - `GET /api/v1/schools/{schoolId}/programs|periods|actors|configurations`
  - `GET /api/v1/programs/{programId}/cohorts|courses`
  - `GET /api/v1/cohorts/{cohortId}/blocks|enrollments`
  - `GET /api/v1/class-sessions/{id}/attendance`, `GET /api/v1/academic-actors/{actorId}/attendance|enrollments`
  - `GET /api/v1/persons/{personId}/biometric-cases`, `GET /api/v1/justifications/{id}/documents`
- IDs: cross-context references are **UUID** (`personId`, `userId`, `academicActorId`, `requestedBy`, `generatedBy`); local catalogs are **INT** (`roleId`, `permissionId`, `environmentId`, `schoolId`) or **BIGINT** (`cohortId`, `scheduleBlockId`, `classSessionId`, `attendanceRecordId`, `alertId`); small catalogs **SMALLINT** (`actorTypeId`, `alertTypeId`). `cityId` removed — external `countriesnow.space` string id. Never expose DB sequences directly without schema.
- Serialization note: DBML `INT/BIGINT` is authoritative for the model. TS services (`03,07`) serialize numeric ids as JSON strings via Zod/Drizzle; Java/Go use JSON numbers. Both map to the same DBML integer — do not change the model to match one serialization.

## 4. HTTP methods

| Method | Use | Success |
|--------|-----|---------|
| GET | Read single / list / nested | 200 |
| POST | Create / action (`/open`, `/bulk`, `/verify`, `/identify`, `/enroll`) | 201 (200 for verify/identify/evaluate) |
| PUT | Full/partial update (Java + TS both accept partial on PUT in this codebase) | 200 (Java returns 204 on some updates) |
| PATCH | Status/review transition | 200 |
| DELETE | Remove / deactivate (soft-delete via `deleted_at` except biometric deactivate) | 204 |

Real deviations documented per contract (e.g. identity `PUT /users/{id}` → 204; biometric `DELETE /facial/{person_id}` → 204).

## 5. Pagination, filtering, sorting

Canonical (see `pagination.md`):

- Java `01` identity: `?page=1&limit=20` (1-based) → `PageResponse {data,meta{page,limit,total,totalPages}}`.
- Other Java services (`02,04,05`): filter-only lists (`?schoolId`, `?cohortId`, `?classSessionId`, `?status`, …) returning bare arrays — no pagination yet; add `?limit&offset` + envelope for new endpoints.
- TS services (`03,07,09`): `?limit=20&offset=0`, cap 100, response header `x-total-count`; `09` list envelope is `{data,total,limit,offset}`; `03/07` return bare array + header.
- Go service (`08`): `?limit&offset` (alert-types default 50/0), filters `?actorId&?typeId&?resolved|?status=open|resolved|pending`.
- Python service (`06`): no pagination (single-template or small history).
- Mobile legacy pattern `?_limit&?_sort&_order` is JSON-Server style and MUST NOT be added to new backend endpoints; backend uses `limit/offset` (identity: `page/limit`).

## 6. Standard responses

| Code | When | Body |
|------|------|------|
| 200 | Success with body | Resource / list |
| 201 | Created | Created resource (+ `Location` where Java/TS set it) |
| 204 | Success without body | Empty |
| 400 | Validation (`Zod`, `Bean Validation`, `Pydantic`, `Gin binding`) | `ErrorResponse` |
| 401 | Missing/invalid/expired session bearer (checked first by Kong's `pre-function`, then by `AuthTokenFilter` in the Java MSs) | `ErrorResponse` |
| 403 | Authenticated but RBAC denies (`GET /auth/evaluate → allowed:false`, or required permission missing) | `ErrorResponse` |
| 404 | Not found (`SchoolNotFound`, `no active facial template`, `no match found`) | `ErrorResponse` |
| 409 | Conflict/duplicate (email exists, person document exists) | `ErrorResponse` |
| 422 | Semantic unprocessable (Python biometric payload) | `ErrorResponse` |
| 500 | Server error | `ErrorResponse` |

## 7. Error format

Canonical envelope is `ErrorResponse` in `contracts/openapi/_shared.yaml`:

```json
{
  "error": "VALIDATION_ERROR",
  "message": "Invalid input data",
  "details": [{ "field": "email", "message": "Invalid email format" }],
  "traceId": "550e8400-e29b-41d4-a716-446655440000"
}
```

Per-stack realities (see `error-catalog.md`):
- Java 01: `{error,status,message,path,timestamp}` via `GlobalExceptionHandler`.
- Java 05: ISO-9001 `{error:{code,message,correlationId,timestamp,isoClause,severity}}` e.g. `ISO-8.2-VAL-001`.
- TS 03/07/09: `{error:BadRequest|NotFound|InternalError,message,details?,timestamp}` + `x-total-count` on lists.
- Python 06: `{error:NotFound|BadRequest|InternalError,message,path,timestamp}` + `422 {error:BadRequest,message:"Invalid biometric payload"}` + headers `x-request-id,x-process-time-ms`.
- Go 08: `{error:string}` with `ErrValidation→400,ErrNotFound→404,ErrDuplicate→409`; `NoRoute→404`, `NoMethod→405`.

Always include `traceId`/`correlationId`/`x-request-id` when available; never leak stack traces.

## 8. Audit and soft-delete

All Postgres tables carry `created_at,updated_at,deleted_at,created_by,updated_by,deleted_by,row_version`. Exposed in contracts as `createdAt,updatedAt,rowVersion` (and `deletedAt` where relevant). Deletes are soft except biometric template deactivation (`is_active=false`) and in-memory quality stores.

## 9. Security

- All inter-service and frontend traffic goes through Kong `:8080`. See `authentication.md`.
- Auth has **two layers**: the edge and the MS. Kong has no `jwt` plugin and no consumers, but a
  global `pre-function` rejects every non-public route without `Authorization: Bearer <sessionId>`
  (opaque UUID from `POST /api/v1/auth/login`) by introspecting identity — public at the edge are
  health probes, `OPTIONS`, `login`, `logout`, `refresh`, `forgot-password`, `verify-code`,
  `reset-password` and `GET /api/v1/auth/me`. Behind that, every service repeats the check on its
  own and adds RBAC (`GET /api/v1/auth/evaluate`) — Java (`01,02,04,05`) in `AuthTokenFilter`, TS
  (`03,07,09`) in `registerAuthGuard`, Python (`06`) in `require_permission`, Go (`08`) in
  `AuthMiddleware`; bootstrap routes (`me`, `GET /sessions/*`) and health/docs are public there
  too. Four separate implementations of the same contract, not a gap (`risks.md` R-003,
  corrected 2026-10-11).
- Local service-to-service calls use UUID cross-references, never cross-schema FKs.
- CORS: global Kong plugin `http://localhost:3000,http://localhost:5173,http://localhost:8090`;
  the identity service also declares `*` on its own plugin for local dev. TLS terminates at Kong in prod.

## 10. Health and quality probes

Every service exposes (no auth):
- `GET /health`, `GET /api/v1/health` (Java also `/api/health`) → `{status,service,version,…}`.
- Quality/ISO probes where implemented: `GET /health/quality`, `GET /api/v1/quality/report` (02,03,04,05,07,08).

## 11. Contract-first checklist (before coding)

1. Add/extend path in `contracts/openapi/<service>.yaml` with `operationId`, `tags`, request/response schemas, `400/401/403/404/500`.
2. Reuse `_shared.yaml` (`ErrorResponse`, `PaginatedMeta`, `UUID`, `Timestamp`, `PageParam/LimitParam/OffsetParam`, `BadRequest/Unauthorized/Forbidden/NotFound/InternalError`).
3. Add Kong route in `kong.yml` matrix + `api-gateway.yaml` + `endpoint-matrix.md`.
4. Add frontend row in `frontend-integration.md` if consumed by Web/Mobile.
5. Add contract test pointer in `11-quality/testing-strategy.md` (consumer-driven).

## 12. Correlations

| Upstream | This doc |
|----------|----------|
| `04-requirements/functional.md` | Each FR maps to endpoints in `endpoint-matrix.md` |
| `06-data/domains/*.md` + `../database/*` | Schemas mirror tables/columns |
| `05-architecture/*`, `09-microservices/*` | Hexagonal ports mirror HTTP adapters |
| Contracts here | Implemented by `back-end/0*` + routed by `99-api-gateway` + consumed per `frontend-integration.md` |
