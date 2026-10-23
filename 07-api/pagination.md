# Pagination, Filtering & Sorting — FaceAttendEDU

> Decisions 2026-10-03 (updated 2026-10-05): new endpoints MUST use `?limit&offset` + `x-total-count`; Quality `{data,total,limit,offset}` stays as documented exception. **`01-ms-identity` already implements `?page&limit` (1-based) + `PageResponse {data,meta}` and that is its canonical contract.** Frontends MUST use `limit/offset` (identity: `page/limit`); `?_limit` is Mobile legacy (JSON-Server) and MUST NOT be added to new backend endpoints.

## 1. Canonical query params

| Param | Type | Default | Max | Used by |
|-------|------|---------|-----|---------|
| `limit` | int 1..100 | 20 | 100 | TS `03/07/09`, Go `08`, new Java lists |
| `offset` | int ≥0 | 0 | — | same as above |
| `page` | int ≥1 | 1 | — | **canonical for identity `01`** (`GET /users|/persons|/sessions?page&limit` → `meta{page,limit,total,totalPages}`); not understood by the other services |

Reusable in `_shared.yaml`: `LimitParam`, `OffsetParam`, `PageParam`.

## 2. Per-stack reality

### Java (`01,02,04,05`)
- `01` identity: `?page=1&limit=20` → `PageResponse {data,meta{page,limit,total,totalPages}}` on
  `/users`, `/persons`, `/sessions`.
- `02/04/05`: lists filter in memory, no envelope and no pagination yet:
  - `GET /api/v1/environments?schoolId=`
  - `GET /api/v1/schedule-blocks?cohortId&environmentId&instructorActorId&courseId`
  - `GET /api/v1/class-sessions?scheduleBlockId`
  - `GET /api/v1/attendance-records?classSessionId&academicActorId&attendanceStatus`
  - `GET /api/v1/justifications?status&attendanceRecordId`
  - `GET /api/v1/supporting-documents?justificationId`
- There is **no** `/api/v1/cities` endpoint: the `city` table was dropped by Liquibase
  (`008-drop-city-table.yaml`) and no controller remains; use the external `countriesnow.space` API.

### TypeScript (`03-academic`, `07-configuration`, `09-quality`)
- `?limit=20&offset=0`, cap 100 (`routes.ts`), response header `x-total-count: <total>`.
- `03/07` return bare sliced array; `09` returns envelope `{data,total,limit,offset}` (`09/routes.ts:75-84`).
- Filters are exact-match: `?name=` (configurations), `?status=` / `?updateStatus=` / `?enrollmentStatus=` (cases/enrollments), `?schoolId/?programId/?cohortId/?actorId` (academic), `?service&?status&?projectId&?processId` (quality).

### TypeScript (`08-notification`)
- `GET /api/v1/alert-types?limit=50&offset=0`; `GET /api/v1/alerts?actorId&typeId&resolved|status&limit&offset`.
- `status=open|pending|resolved` maps to `resolved` bool; response envelopes `{alerts:[]}` / `{alert_types:[]}` (not `data/meta`).

### Python (`06-biometric`)
- No pagination. Single active template (`GET /facial/{person_id}`), small history (`/history`), 1:1 verify and 1:N identify are POST with body.

## 3. Sorting

- Only Mobile legacy uses `?_sort=raised_at&_order=desc` (alerts dashboard). Backend has no `sort/order` params today.
- New endpoints SHOULD accept `?sort=<field>&order=asc|desc` and document allowed fields; default stable order is `created_at ASC, id ASC`.

## 4. Frontend guidance

- Web: `api/apiClient.js` reads the page envelope (`isPageEnvelope`/`unwrapPage`) and exposes
  `fetchAllPages(path, query)`; identity calls pass `page/limit`, TS/Go calls `limit/offset`.
- Mobile: identity lists use `{page:1,limit:…}`; two viewmodels still send JSON-Server
  `?_limit=100|500` (`useManageEnvironmentViewModel`, `useAttendanceReportViewModel`) — migrate
  them to `?limit=&offset=`; keep `_sort/_order` client-side until backend supports it.
- Always send `limit` explicitly; never assume default page size.

## 5. Examples

```
# identity paged (via gateway)
GET http://localhost:8000/api/v1/persons?page=1&limit=20
# → 200 {"data":[...],"meta":{"page":1,"limit":20,"total":57,"totalPages":3}}

# TS paginated (via gateway)
GET http://localhost:8000/api/v1/schools?limit=20&offset=40
# → 200 [...] + x-total-count: 142

# Java filtered
GET http://localhost:8000/api/v1/attendance-records?classSessionId=12&attendanceStatus=Absent

# Go filtered + paginated
GET http://localhost:8000/api/v1/alerts?actorId=77&status=open&limit=20&offset=0
# → 200 {"alerts":[...]}
```
