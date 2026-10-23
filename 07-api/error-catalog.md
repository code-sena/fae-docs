# Error Catalog — FaceAttendEDU

> Decisions 2026-10-03: `{error,message}` is the minimal contract in `_shared.yaml` validated by `11-quality` tests; per-stack fields stay as allowed extensions. Canonical envelope is `ErrorResponse` (`_shared.yaml`). Per-stack wire formats below are real (from exception handlers).

## 1. Canonical envelope

```json
{
  "error": "VALIDATION_ERROR",
  "message": "Invalid input data",
  "details": [{ "field": "email", "message": "Invalid email format" }],
  "traceId": "550e8400-e29b-41d4-a716-446655440000"
}
```

| `error` code | HTTP | When |
|--------------|------|------|
| `VALIDATION_ERROR` / `BadRequest` | 400 | Zod / Bean Validation / Pydantic validation fails |
| `UNAUTHORIZED` | 401 | Missing/invalid/expired session bearer (rejected first by Kong's `pre-function`, then by the Java `AuthTokenFilter`) |
| `FORBIDDEN` | 403 | `GET /auth/evaluate → allowed:false` |
| `NOT_FOUND` / `NotFound` | 404 | `SchoolNotFound`, `no active facial template`, `no match found` |
| `CONFLICT` / `Duplicate` | 409 | Email/document already exists |
| `UNPROCESSABLE_ENTITY` | 422 | Python biometric semantic error |
| `INTERNAL_ERROR` / `InternalError` | 500 | Unhandled |

## 2. Per-stack wire formats (real)

### Java 01-identity — `GlobalExceptionHandler.java:21-28`
```json
{ "error": "NotFound", "status": 404, "message": "…", "path": "/api/v1/persons/99", "timestamp": "2024-01-15T10:30:00Z" }
```

### Java 05-attendance — ISO-9001 `GlobalExceptionHandler.java:93-112`
```json
{ "error": { "code": "ISO-8.2-VAL-001", "message": "…", "correlationId": "…", "timestamp": "…", "isoClause": "8.2", "severity": "HIGH" } }
```
Codes seen: `ISO-8.2-VAL-001` (validation), `ISO-8.5-INT-001` (integrity), `ISO-10.2-NC-001` (nonconformity).

### TS 03/07/09 — `main.ts` error handler + `qualityMiddleware`
```json
{ "error": "BadRequest", "message": "schoolId is required", "details": [...], "timestamp": "…" }
{ "error": "NotFound", "message": "School not found", "timestamp": "…" }
{ "error": "SchoolNotFound", "message": "…", "timestamp": "…" }
```
Lists also send `x-total-count` header even on 200 with empty array.

### Python 06 — `main.py:31-56`
```json
{ "error": "NotFound", "message": "no active facial template", "path": "/api/v1/biometric/facial/…", "timestamp": "…" }
{ "error": "BadRequest", "message": "Invalid biometric payload", "details": [...], "path": "…" }
```
Plus headers `x-request-id`, `x-process-time-ms`, `x-content-type-options: nosniff`. Threshold `SIMILARITY_THRESHOLD=0.85`.

### TypeScript 08 — `infrastructure/http/routes.ts`, `main.ts:46-61`
```json
{ "error": "BadRequest", "message": "Invalid alert payload", "details": {...}, "timestamp": "…" }
{ "error": "NotFound" }
{ "error": "Forbidden", "message": "Forbidden: outside your scope", "timestamp": "…" }
{ "error": "NotFound", "message": "Route GET /x not found", "timestamp": "…" }
```
`POST /api/v1/emails` uses its own codes (`FORBIDDEN` for a bad internal token, `VALIDATION_ERROR`, `DELIVERY_FAILED` 502). The global error handler maps `statusCode ≥ 400` through `errorNameFor` and hides raw messages for 5xx (`Unexpected internal error`). Rewritten from Go on 2026-10-18 (ADR-021): the old `{error:"validation failed: …"}` / `route not found` bodies no longer exist.

### Mobile client — `apiClient.js:70-83`
`ApiError(status,message,data)`: `!ok → throw ApiError(status, result.message||"Error <status>")`; `AbortError→408 timeout`; network → `0`.

## 3. Frontend handling (real)

- Mobile `useLoginViewModel.js:49-59`: `0/408→connectionError`, else `ApiError.message`, else `invalidCredentials`.
- Web `viewmodels/useAuthViewModel.js` → `AuthContext.login()` → `loginErrorMessage(err)` in
  `context/AuthContext.js:24-28`: `400/401→"Credenciales incorrectas"`, `status 0→` network
  message, else `err.message`. (The old `useLoginViewModel.ts` mock path no longer exists.)
- Alerts/verification: `useCustomAlert{showSuccess,showError,showConfirm}`; `VerificationError{INVALID_CODE,EXPIRED_CODE,NO_CODE,TIMEOUT,GENERIC}` with 15s timeout.

## 4. Rules

1. Always return `ErrorResponse`-compatible body (at minimum `{error,message}`); include `details[]` for 400/422, `traceId/correlationId/x-request-id` when available.
2. Never leak stacks, SQL, or secrets. Log full error server-side with correlation id.
3. Document every `400/401/403/404/409/422/500` per operation in OpenAPI (reuse `_shared.yaml` responses).
4. Mobile/Web MUST surface `message`, not `error`, to users; use i18n keys where available.
