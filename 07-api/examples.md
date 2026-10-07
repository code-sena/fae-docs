# API Examples — FaceAttendEDU

> Copy-paste `curl` via Kong (`:8080`). Assumes a session id in `$TOKEN` — Kong has **no jwt
> plugin/consumers**, so `$TOKEN` is the opaque `sessionId` issued at login and sent as
> `Authorization: Bearer`. Java services (01/02/04/05) reject requests without it; the
> TS/Python/Go services do not check it yet.

## 0. Token

```bash
# login (public): identifier may be an email OR a username
curl -s -X POST http://localhost:8080/api/v1/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"identifier":"admin@example.com","password":"Secret123!"}'
# {"email":"…"} / {"username":"…"} are accepted aliases of {"identifier":"…"}
# → {"sessionId":"…","userId":"…","sessionStatus":"Active",…}
# wrong password → 401 {"message":"Invalid email or password"}

export TOKEN=<sessionId>
```

## 1. Identity

```bash
# cities: deprecated in the relational model — prefer the external API
curl -s 'https://countriesnow.space/api/v0.1/countries/cities' \
  -H 'Content-Type: application/json' -d '{"country":"Colombia"}'
# the legacy ms-identity /cities CRUD still exists behind Kong (see endpoint-matrix.md §1)
curl -s 'http://localhost:8080/api/v1/persons?page=1&limit=20' -H "Authorization: Bearer $TOKEN"
curl -s http://localhost:8080/api/v1/persons/<uuid> -H "Authorization: Bearer $TOKEN"
```

## 2. Authorization (session bearer)

```bash
curl -s http://localhost:8080/api/v1/roles -H "Authorization: Bearer $TOKEN"
curl -s "http://localhost:8080/api/v1/auth/evaluate?userId=<uuid>&permission=attendance.record:read" \
  -H "Authorization: Bearer $TOKEN"
# → {"userId":"…","permission":"attendance.record:read","allowed":true}
```

## 3. Academic (no MS-side auth yet)

```bash
curl -s 'http://localhost:8080/api/v1/schools?limit=20&offset=0' -H "Authorization: Bearer $TOKEN"
curl -s -X POST http://localhost:8080/api/v1/schools -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' -d '{"code":"COL001","name":"Colegio San José","cityId":"CO-BOG","cityName":"Bogotá","district":"Cundinamarca","country":"Colombia"}'
curl -s http://localhost:8080/api/v1/programs/<id>/cohorts -H "Authorization: Bearer $TOKEN"
```

## 4. Scheduling (session bearer)

```bash
curl -s 'http://localhost:8080/api/v1/schedule-blocks?cohortId=1' -H "Authorization: Bearer $TOKEN"
curl -s -X POST http://localhost:8080/api/v1/class-sessions/<id>/open \
  -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' -d '{}'
```

## 5. Attendance (session bearer)

```bash
curl -s 'http://localhost:8080/api/v1/attendance-records?classSessionId=12' \
  -H "Authorization: Bearer $TOKEN"
curl -s -X POST http://localhost:8080/api/v1/attendance-records/bulk \
  -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  -d '[{"classSessionId":12,"academicActorId":77,"attendanceStatus":"Present","captureMethod":"FACIAL","matchScore":0.97}]'
curl -s -X PATCH http://localhost:8080/api/v1/justifications/5/review \
  -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  -d '{"reviewStatus":"Approved","resolutionNotes":"Valid supporting document"}'
```

## 6. Biometric (no MS-side auth yet)

```bash
curl -s -X POST http://localhost:8080/api/v1/biometric/facial/verify \
  -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  -d '{"person_id":"p1","encoding":[0.1,0.2,0.3]}'
# → {"match":true,"score":0.93} (threshold 0.85)
```

## 7. Configuration (no MS-side auth yet)

```bash
curl -s 'http://localhost:8080/api/v1/biometric-update-cases?status=Pending&limit=20&offset=0' \
  -H "Authorization: Bearer $TOKEN"
curl -s -X PATCH http://localhost:8080/api/v1/biometric-update-cases/<id>/review \
  -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  -d '{"updateStatus":"Approved"}'
```

## 8. Notification (no MS-side auth yet)

```bash
curl -s 'http://localhost:8080/api/v1/alerts?actorId=77&status=open&limit=20&offset=0' \
  -H "Authorization: Bearer $TOKEN"
# → {"alerts":[…]}
curl -s -X PATCH http://localhost:8080/api/v1/alerts/10/resolve \
  -H "Authorization: Bearer $TOKEN"
```

## 9. Quality (no MS-side auth yet)

```bash
curl -s http://localhost:8080/api/v1/quality/characteristics -H "Authorization: Bearer $TOKEN"
curl -s 'http://localhost:8080/api/v1/quality/evaluations?service=attendance&limit=20&offset=0' \
  -H "Authorization: Bearer $TOKEN"
# → {"data":[…],"total":…,"limit":20,"offset":0}
```

## 10. Health (public)

```bash
# the gateway only routes /health/quality (proxied to ms-academic:8083) — there is no GET /health at :8080
curl -s http://localhost:8080/health/quality
# probe services directly on their own ports
curl -s http://localhost:8081/api/v1/health
curl -s http://localhost:8083/health
curl -s http://localhost:8083/health/quality
```
