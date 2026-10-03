# Frontend Integration — FaceAttendEDU APIs

> Decisions 2026-10-03: canonical base is `http://localhost:8080/api/v1` via Kong for Web and Mobile; default is `requiresAuth:true` with refresh-on-401. Web mocks stay as fallback only.

> How `front-end/Web` and `front-end/Mobile` really consume the backend.
> Sources: `front-end/Web/src/**`, `front-end/Mobile/src/**`.

## 1. TL;DR

- **Web (`front-end/Web`, Expo Web + RN-Web): does NOT consume the real backend.** 100% mock/local. `api/apiClient.ts` is a stub, `constants/urls.ts` (`http://10.3.234.12:8080/` + paths) has zero imports. Auth is dual mock (`AuthService` dict + `AuthContext` delay). Migration to Kong `:8080` pending.
- **Mobile (`front-end/Mobile`, Expo 57): DOES consume the backend** (JSON-Server style + Auth + Face). Centralized in `api/apiClient.js` (`fetch`, 15s timeout, `ApiError`). All services use `requiresAuth:false` today (debt: never sends `Bearer`).

## 2. Base URLs

| App | Config | Real value |
|-----|--------|------------|
| Web | `services/constants/urls.ts:1` | `http://10.3.234.12:8080/` (dead, no imports) |
| Web | env / `app.json` | No `.env`, no `VITE_API_URL`, no `EXPO_PUBLIC_*` |
| Web i18n | `LibreTranslateProvider.ts:37` | `https://translate.argosopentech.com` (external, non-business) |
| Web i18n | `RestTranslationProvider.ts:34` | `http://localhost:5000` (external) |
| Mobile | `config/env.js:16-36` | `EXPO_PUBLIC_API_URL` if set, else `http://<host>:3000/`, fallback `http://10.3.234.223:3000/` |
| Mobile | `.env:6` | `EXPO_PUBLIC_AUTH_MODE=legacy` (only var) |
| Mobile face | `UpdatePhotoScreen.js:28` | `EXPO_PUBLIC_FACE_URL || ${API_BASE_URL}face/register` (multipart, no auth) |
| Mobile countries | `api/apiCountry.js:1` | `https://countriesnow.space/api/v0.1` (external) |
| Prod target | — | `http://localhost:8080/api/v1/...` (Kong) for both apps |

## 3. Auth flows

### Mobile (real, multistep)
1. `GET person?email` → `GET app_user?person_id` → `GET user_role?user_id` → loop `GET role?role_id` → `roleNames[]`.
2. `POST user_session {user_id}` → `token = session.token||session_token||session_id||id`.
3. `AuthResponse{token,user{…roles,person}}` → `TokenStorage.saveToken` (jwtDecode exp, `AsyncStorage auth_token`).
4. `getHighestRole(roles)` → `AsyncStorage userEmail` + theme/language by role → `onLogin(role,token)`.
5. Refresh: `POST user_session/refresh {refresh_token}`. Logout: `DELETE user_session/{id}` (viewmodels today only clear local state).
6. Recovery: `POST auth/forgot-password|verify-code|reset-password` (15s timeout, `VerificationError`, `MAX_CODE_ATTEMPTS=5`, 30s lockout, 600s expiry, 60s resend).

### Web (mock)
- Path A (`AuthService`): dict `admin/teacher/student@example.com:123456` → hardcoded JWT; `register` fabricates `eyJ…`.
- Path B (`AuthContext`): `delay 800ms → mockAppUsers.find(email)`; `fetch("/api/auth/login")` commented out. Session in `localStorage @faceattend:session_user`.
- No refresh, no `Bearer`, no interceptors.

## 4. Endpoint usage

### Web: 0 business HTTP calls
All screens read `useAppData()` / `mockData.ts` / `models/data/*Storage` (local CRUD). `FaceRegistrationModal` does local cropping, no `POST face/*`.

### Mobile: generic services + N+1 in viewmodels
`getAll/getById/create/update/delete` pattern over `app_user,person,user_session,academic_actor,attendance_record,justification,justification_type,course,program,school,academic_period,cohort,enrollment,environment,schedule_block,class_session,alert,alert_type,academic_configuration,security_configuration,biometric_update_case,role,permission` + `face/register` multipart + external `countriesnow`. Removed: `city,audit_log,error_log` (not relational tables); `blood_type,document_type` are person fields, not resources.

ViewModels with direct calls (`request()` / `fetch`):
- `useDashboardViewModel`: `GET attendance_record?_limit=100`, `GET class_session?class_session_id=`, `GET schedule_block?schedule_block_id=`, `GET course?course_id=`, `GET alert_type`, `GET alert?_sort=raised_at&_order=desc&_limit=5`, `GET academic_actor?academic_actor_id=`, `GET person?person_id=`, `GET justification?review_status=Pending`.
- `useDisplayingAttendanceViewModel`: `GET attendance_record?_limit=200` + N+1 enrichment (actor→person, block→course→environment).
- `useAttendanceReportViewModel`: `GET attendance_record?_limit=500` + `POST attendance_report {role,attendance_type,people}` (no backend endpoint — `attendance_report` is a derived JSONB projection with no FKs, outside the 3NF core; gap recorded).
- `useAddJustifyViewModel`: `POST justification {attendance_record_id,justification_type_id,reason,submitted_at,review_status:Pending,attachment{…}}`.
- `useManageUsersViewModel`: `GET app_user?_limit=50` + `GET user_role?user_id=` + `GET person?person_id=`.
- `UpdatePhotoScreen`: `POST face/register` multipart.

## 5. DTOs, errors, pagination

- DTOs: `BaseModel{created_at,updated_at,deleted_at,created_by,updated_by,deleted_by,row_version}` + `fromApi/toApi` snake_case; `AuthRequest/AuthResponse`, `AttendanceRecord{Present/Absent/Late/Justified}`, `captureMethod{FACIAL/MANUAL/IOT/IMPORT}`, canonical roles `SUPER_ADMIN/SCHOOL_ADMIN/INSTRUCTOR/STUDENT` (legacy code uses `Administrator/Teacher/Student` and Spanish `Administrador/Docente/Estudiante` — migrate).
- Errors: `ApiError(status,message,data)`; login maps `0/408→connectionError`; viewmodels use `useCustomAlert`; `LanguageService` falls back to `DEFAULT_LANGUAGES`; `CountryService` `{success,data/error}`.
- Pagination: only `?_limit (+_sort/_order on alerts)`, no `page/offset`. Migrate to backend `?limit&offset`.

## 6. Migration checklist (Web + Mobile → Kong)

1. Point `API_BASE_URL` to `http://localhost:8080` (env `EXPO_PUBLIC_API_URL` / Web `.env`).
2. Always send `Authorization: Bearer` (`requiresAuth:true`) + retry with refresh on 401.
3. Replace `?_limit` with `?limit&offset` and read `x-total-count` / `{data,total,limit,offset}`.
4. Remove client-side N+1 using nested reads (`/class-sessions/{id}/attendance`, `/cohorts/{id}/blocks`, …).
5. Implement `POST attendance_report` in backend or remove the Mobile call.
6. Send `face/register` to `POST /api/v1/biometric/facial/enroll|verify` with JWT (no anonymous multipart).
