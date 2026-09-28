# Pagination, Filtering & Sorting — FaceAttendEDU

> Real behaviors per stack. Frontends MUST use `limit/offset`; `?_limit` is Mobile legacy (JSON-Server) and MUST NOT be added to new backend endpoints.

## 1. Canonical query params

| Param | Type | Default | Max | Used by |
|-------|------|---------|-----|---------|
| `limit` | int 1..100 | 20 | 100 | TS `03/07/09`, Go `08`, new Java lists |
| `offset` | int ≥0 | 0 | — | same as above |
| `page` | int ≥1 | 1 | — | docs-only alias; backend implements `offset`, gateway does not translate |

Reusable in `_shared.yaml`: `LimitParam`, `OffsetParam`, `PageParam` (docs alias).

## 2. Per-stack reality

### Java (`01,02,04,05`)
- `GET /api/v1/cities` removed (city table dropped; use external `countriesnow.space` API).
- Lists filter in memory, no envelope:
  - `GET /api/v1/environments?schoolId=`
  - `GET /api/v1/schedule-blocks?cohortId&environmentId&instructorActorId&courseId`
  - `GET /api/v1/class-sessions?scheduleBlockId`
  - `GET /api/v1/attendance-records?classSessionId&academicActorId&attendanceStatus`
  - `GET /api/v1/justifications?status&attendanceRecordId`
  - `GET /api/v1/supporting-documents?justificationId`

### TypeScript (`03-academic`, `07-configuration`, `09-quality`)
- `?limit=20&offset=0`, cap 100 (`routes.ts`), response header `x-total-count: <total>`.
- `03/07` return bare sliced array; `09` returns envelope `{data,total,limit,offset}` (`09/routes.ts:75-84`).
- Filters are exact-match: `?name=` (configurations), `?status=` / `?updateStatus=` / `?enrollmentStatus=` (cases/enrollments), `?schoolId/?programId/?cohortId/?actorId` (academic), `?service&?status&?projectId&?processId` (quality).

### Go (`08-notification`)
- `GET /api/v1/alert-types?limit=50&offset=0`; `GET /api/v1/alerts?actorId&typeId&resolved|status&limit&offset`.
- `status=open|pending|resolved` maps to `resolved` bool; response envelopes `{alerts:[]}` / `{alert_types:[]}` (not `data/meta`).

### Python (`06-biometric`)
- No pagination. Single active template (`GET /facial/{person_id}`), small history (`/history`), 1:1 verify and 1:N identify are POST with body.

## 3. Sorting

- Only Mobile legacy uses `?_sort=raised_at&_order=desc` (alerts dashboard). Backend has no `sort/order` params today.
- New endpoints SHOULD accept `?sort=<field>&order=asc|desc` and document allowed fields; default stable order is `created_at ASC, id ASC`.

## 4. Frontend guidance

- Web: no pagination today (in-memory filter). When wiring to backend, use `limit/offset` + `x-total-count`.
- Mobile: migrate `?_limit=50|100|200|500` → `?limit=&offset=`; keep `_sort/_order` client-side until backend supports it.
- Always send `limit` explicitly; never assume default page size.

## 5. Examples

```
# TS paginated (via gateway)
GET http://localhost:8080/api/v1/schools?limit=20&offset=40
# → 200 [...] + x-total-count: 142

# Java filtered
GET http://localhost:8080/api/v1/attendance-records?classSessionId=12&attendanceStatus=Absent

# Go filtered + paginated
GET http://localhost:8080/api/v1/alerts?actorId=77&status=open&limit=20&offset=0
# → 200 {"alerts":[...]}
```
