# API Examples — FaceAttendEDU

> Copy-paste `curl` via Kong (`:8080`). Assumes dev JWT in `$TOKEN`
> (`faceattend-admin` HS256 for local). Identity calls omit auth (public).

## 0. Token

```bash
# login (identity public)
curl -s -X POST http://localhost:8080/api/v1/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"username":"admin@example.com","password":"Secret123!"}'
# → {"sessionId":"…","userId":"…","sessionStatus":"Active"}

export TOKEN=<sessionId>
```

## 1. Identity (public)

```bash
curl -s http://localhost:8080/api/v1/cities?limit=20&offset=0
curl -s -X POST http://localhost:8080/api/v1/cities \
  -H 'Content-Type: application/json' -d '{"name":"Medellín","department":"Antioquia"}'
curl -s http://localhost:8080/api/v1/persons/<uuid>
```

## 2. Authorization (jwt)

```bash
curl -s http://localhost:8080/api/v1/roles -H "Authorization: Bearer $TOKEN"
curl -s "http://localhost:8080/api/v1/auth/evaluate?userId=<uuid>&permission=attendance:write" \
  -H "Authorization: Bearer $TOKEN"
# → {"userId":"…","permission":"attendance:write","allowed":true}
```

## 3. Academic (jwt)

```bash
curl -s 'http://localhost:8080/api/v1/schools?limit=20&offset=0' -H "Authorization: Bearer $TOKEN"
curl -s -X POST http://localhost:8080/api/v1/schools -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' -d '{"code":"COL001","name":"Colegio San José","cityId":1}'
curl -s http://localhost:8080/api/v1/programs/<id>/cohorts -H "Authorization: Bearer $TOKEN"
```

## 4. Scheduling (jwt)

```bash
curl -s 'http://localhost:8080/api/v1/schedule-blocks?cohortId=1' -H "Authorization: Bearer $TOKEN"
curl -s -X POST http://localhost:8080/api/v1/class-sessions/<id>/open \
  -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' -d '{}'
```

## 5. Attendance (jwt)

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

## 6. Biometric (jwt)

```bash
curl -s -X POST http://localhost:8080/api/v1/biometric/facial/verify \
  -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  -d '{"person_id":"p1","encoding":[0.1,0.2,0.3]}'
# → {"match":true,"score":0.93} (threshold 0.85)
```

## 7. Configuration (jwt)

```bash
curl -s 'http://localhost:8080/api/v1/biometric-update-cases?status=Pending&limit=20&offset=0' \
  -H "Authorization: Bearer $TOKEN"
curl -s -X PATCH http://localhost:8080/api/v1/biometric-update-cases/<id>/review \
  -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  -d '{"updateStatus":"Approved"}'
```

## 8. Notification (jwt)

```bash
curl -s 'http://localhost:8080/api/v1/alerts?actorId=77&status=open&limit=20&offset=0' \
  -H "Authorization: Bearer $TOKEN"
# → {"alerts":[…]}
curl -s -X PATCH http://localhost:8080/api/v1/alerts/10/resolve \
  -H "Authorization: Bearer $TOKEN"
```

## 9. Quality (jwt)

```bash
curl -s http://localhost:8080/api/v1/quality/characteristics -H "Authorization: Bearer $TOKEN"
curl -s 'http://localhost:8080/api/v1/quality/evaluations?service=attendance&limit=20&offset=0' \
  -H "Authorization: Bearer $TOKEN"
# → {"data":[…],"total":…,"limit":20,"offset":0}
```

## 10. Health (public)

```bash
curl -s http://localhost:8080/health
curl -s http://localhost:8081/api/v1/health
curl -s http://localhost:8084/health/quality
```
