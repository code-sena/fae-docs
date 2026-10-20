# Frontend Integration — FaceAttendEDU APIs

> Decisions 2026-10-03 (updated 2026-10-05, commits `c4fad72`/`e1f6d71`): canonical base is
> `http://localhost:8000/api/v1` via Kong for Web **and** Mobile; both apps call the real backend
> and send `Authorization: Bearer <sessionId>`. Mocks survive only as fallback data
> (`EXPO_PUBLIC_API_MOCK=1` / API failure).

> How `front-end/Web` and `front-end/Mobile` really consume the backend.
> Sources: `front-end/Web/src/**`, `front-end/Mobile/src/**`, `COMPOSE.md`.

## 1. TL;DR

- **Web (`front-end/Web`, Expo Web + RN-Web): consumes the real backend.**
  `src/api/apiClient.js` (`request`, `fetchAllPages`, automatic `Bearer`, 15s timeout,
  `ApiError`) + `src/api/endpoints.js` (verified catalog of all `/api/v1` routes) +
  `src/services/api/{auth,academic,authorization,configuration,notification,quality,
  scheduling,referenceData}Api.js`. `context/AuthContext.js` performs the real login
  (`POST /api/v1/auth/login`). `models/data/mockData.js` is a **fallback** when the API is
  unreachable or `EXPO_PUBLIC_API_MOCK=1`; the old `constants/urls.ts`, `services/AuthService.ts`
  and `rolesHierarchy.ts` were deleted.
- **Mobile (`front-end/Mobile`, Expo): consumes the real backend** through Kong.
  Centralized in `api/apiClient.js` (`fetch`, 15s timeout, `ApiError`, `requiresAuth` defaults to
  **true** and the `Bearer` token is attached whenever a session exists) + `api/backend.js`
  (`backendGet`, snake_case mapping, optional cache). ViewModels call `api/v1/...` REST paths
  directly.

## 2. Base URLs

| App | Config | Real value |
|-----|--------|------------|
| Web | `config/env.ts getApiBaseUrl()` | `EXPO_PUBLIC_API_URL` (build time) → `http://localhost:8000`; runtime override `window.__FACEATTEND_API_URL__` |
| Web | `config/env.ts API_MODE` | `live` unless `EXPO_PUBLIC_API_MOCK=1` |
| Web i18n | `LibreTranslateProvider.ts:37` | `https://translate.argosopentech.com` (external, non-business) |
| Web i18n | `RestTranslationProvider.ts:34` | `http://localhost:5000` (external) |
| Mobile | `config/env.js resolveBaseUrl()` | `EXPO_PUBLIC_API_URL` → Metro `hostUri` host → `10.0.2.2` (Android emulator) → `10.3.233.33` LAN default, all on **`:8000` (Kong)**; `AUTHZ/ACADEMIC/ATTENDANCE/SCHEDULING/NOTIFY_BASE_URL` are aliases of the same gateway |
| Mobile | `.env` | `EXPO_PUBLIC_API_URL` / `EXPO_PUBLIC_AUTH_MODE` (only vars consumed) |
| Mobile face | `UpdatePhotoScreen.js` | `EXPO_PUBLIC_FACE_URL \|\| ${API_BASE_URL}face/register` (multipart, no auth header) |
| Mobile countries | `api/apiCountry.js:1` | `https://countriesnow.space/api/v0.1` (external) |
| Prod target | — | gateway behind TLS, `https://<host>/api/v1/...` for both apps |

Docker: the compose stack publishes the gateway on `KONG_PROXY_PORT` (8000) and the Web app on
**8090** (`FRONTEND_PORT`), so browser CORS is permissive (Kong `cors` origins are `["*"]`).

## 3. Auth flows (both real)

### Mobile (`services/AuthService.js`)

1. `POST api/v1/auth/login {email,password}` — the backend resolves the email → user through
   `identity.person.email` (no client-side person/app_user/user_role resolution any more).
   400/401 → "Credenciales inválidas".
2. `saveToken(sessionId)` into `expo-secure-store` (AsyncStorage on web) **before** any
   authenticated call.
3. In parallel: `GET api/v1/users/{id}/roles → roleName[]` and profile
   (`GET users/{id}` → `persons/{id}`; fallback `GET users?page&limit`).
4. `GET api/v1/auth/evaluate?userId&permission=attendance.record:read` — fail-closed: without a
   real grant there is no login.
5. `AuthResponse{token,user{…roles,person}}` → `getHighestRole()` (`admin > teacher > student`) →
   theme/language by role.
6. Refresh: the backend has `POST api/v1/auth/refresh {sessionId}` (returns a new session id and
   closes the previous one), but Mobile's `refreshToken()` still throws "Sesión no renovable:
   vuelve a iniciar sesión" — the client never calls it.
   Logout: `POST api/v1/auth/logout?sessionId` (best-effort) + clear storage.
7. Recovery: `POST auth/forgot-password|verify-code|reset-password`
   (`verificationService.js|passwordService.js`, 15s timeout, `VerificationError`,
   `MAX_CODE_ATTEMPTS=5`, 30s lockout, 600s expiry, 60s resend) — **implemented server-side**
   (`01-ms-identity`, same values: 6-digit code, 10 min TTL, 5 attempts, 60s resend; verified
   2026-10-07); delivery depends on SMTP (`ms-notification`), see §6.

### Web (`context/AuthContext.js` + `services/api/authApi.js`)

- `login({email,password})` → `POST /api/v1/auth/login` → `sessionId` + `userId` required,
  `saveToken(sessionId,{email})` (localStorage `@faceattend:session_user` on web) → loads
  roles/profile from the API → `toUiRole(roleNames)` from `core/utils/backendRoles.js`.
- Session bootstrap on mount: `sessionGet() + getToken()` in parallel; invalid session → cleanup.
- `logout()` clears token + session key. `register()` still validates locally and (until the
  backend exposes a register route) creates the account through the API when available.
- Errors surface through `loginErrorMessage(err)` (401 → invalid credentials, network → gateway
  `:8000` down message).

## 4. Endpoint usage

### Web: storages talk to the API, mocks only on failure

- `models/data/{UserStorage,StudentStorage,FichaStorage,EnvironmentStorage,CourseStorage}.js`
  read through `services/api/referenceData.js` first and fall back to `mockData.js` when the API
  is unavailable (`console.warn "[FaceAttend] load*: API no disponible, uso mock"`).
- `viewmodels/useDashboardViewModel.js` and `useReportsViewModel.js` still import chart series
  from `models/data/mockData.js` (presentation data, not business CRUD).
- `FaceRegistrationModal` does local cropping only (no `POST face/*` from Web).

### Mobile: `backendGet(ENV.X_BASE_URL, 'api/v1/…', params)` in viewmodels

Examples: `useDashboardViewModel` → `class-sessions/{id}`, `schedule-blocks/{id}`,
`courses/{id}`, `alert-types`, `alerts?limit=50`, `academic-actors?limit=2000`,
`attendance-records?page&limit`, `justifications?status=Pending`, `schedule-blocks?
instructorActorId=…`; `useManageEnvironmentViewModel` → `environments`; attendance report /
justification viewmodels → `attendance-records`, `justifications`, `supporting-documents`.

Endpoints still missing on the backend: `POST attendance_report` (derived JSONB projection, no
route yet) and `POST face/register` (Web/Mobile still use the anonymous multipart helper instead
of `POST /api/v1/biometric/facial/enroll|verify`).

## 5. DTOs, errors, pagination

- DTOs: `BaseModel{createdAt…,rowVersion}` + `fromApi/toApi` snake_case; `AuthRequest{email|username}`,
  `AuthResponse`, `AttendanceRecord{Present/Absent/Late/Justified}`,
  `captureMethod{FACIAL/MANUAL/IOT/IMPORT}`.
- Roles: canonical catalog is **`Administrador | Instructor | Aprendiz`**
  (`004-unify-mobile-roles`); `backendRoles.js` also accepts the legacy
  `SUPER_ADMIN/SCHOOL_ADMIN/INSTRUCTOR/STUDENT/DOCENTE/…` names so an un-migrated database still
  maps to `admin|teacher|student`.
- Errors: `ApiError(status,message,data)`; login maps `0/408` to connection errors; viewmodels
  use `useCustomAlert`.
- Pagination: identity lists use **`?page&limit` → `{data,meta}`**; TS/Go use
  `?limit&offset` + `x-total-count`. A few Mobile calls still pass JSON-Server leftovers
  (`_limit:500` in `useAttendanceReportViewModel`, `_limit:100` in `useManageEnvironmentViewModel`)
  — harmless today (ignored by the backends) but MUST be migrated.

## 6. Pending gaps (Web + Mobile → backend)

1. Recovery is implemented server-side, but the e2e email delivery is unverified: with `SMTP_HOST`
   empty, `POST ms-notification:8088/api/v1/emails` answers `202 {status:"skipped"}` and the flow
   stalls — verify SMTP delivery end to end.
2. `POST attendance_report` has no backend route.
3. Remove the `?_limit` JSON-Server leftovers (§5) and keep `page/limit` (identity) or
   `limit/offset` elsewhere.
4. Collapse remaining N+1 enrichment into nested reads
   (`/class-sessions/{id}/attendance`, `/cohorts/{id}/blocks`, `/schools/{id}/actors`, …).
5. Replace anonymous `face/register` with `POST /api/v1/biometric/facial/enroll|verify` +
   `Authorization: Bearer`.
6. Keep sending `Bearer` on every call; 401 → try `POST /api/v1/auth/refresh` once, otherwise
   clear session → login (Mobile does not implement the refresh call yet).
