# Frontend Integration — FaceAttendEDU APIs

> How `front-end/Web` and `front-end/Mobile` really consume the backend.
> Sources: `front-end/Web/src/**`, `front-end/Mobile/src/**`.

## 1. TL;DR

- **Web (`front-end/Web`, Expo Web + RN-Web): NO consume backend real.** 100% mock/local. `api/apiClient.ts` es un stub, `constants/urls.ts` (`http://10.3.234.12:8080/` + paths) tiene cero imports. Auth es mock dual (`AuthService` dict + `AuthContext` delay). Migración pendiente a Kong `:8080`.
- **Mobile (`front-end/Mobile`, Expo 57): SÍ consume backend** (estilo JSON-Server + Auth + Face). Centralizado en `api/apiClient.js` (`fetch`, timeout 15s, `ApiError`). Todos los servicios usan `requiresAuth:false` hoy (deuda: nunca envía `Bearer`).

## 2. Base URLs

| App | Config | Valor real |
|-----|--------|------------|
| Web | `services/constants/urls.ts:1` | `http://10.3.234.12:8080/` (muerta, sin imports) |
| Web | env / `app.json` | Sin `.env`, sin `VITE_API_URL`, sin `EXPO_PUBLIC_*` |
| Web i18n | `LibreTranslateProvider.ts:37` | `https://translate.argosopentech.com` (externo, no negocio) |
| Web i18n | `RestTranslationProvider.ts:34` | `http://localhost:5000` (externo) |
| Mobile | `config/env.js:16-36` | `EXPO_PUBLIC_API_URL` si existe, sino `http://<host>:3000/`, fallback `http://10.3.234.223:3000/` |
| Mobile | `.env:6` | `EXPO_PUBLIC_AUTH_MODE=legacy` (única var) |
| Mobile face | `UpdatePhotoScreen.js:28` | `EXPO_PUBLIC_FACE_URL \|\| ${API_BASE_URL}face/register` (multipart, sin auth) |
| Mobile países | `api/apiCountry.js:1` | `https://countriesnow.space/api/v0.1` (externo) |
| Target prod | — | `http://localhost:8080/api/v1/...` (Kong) para ambas apps |

## 3. Auth flows

### Mobile (real, multistep)
1. `GET person?email` → `GET app_user?person_id` → `GET user_role?user_id` → loop `GET role?role_id` → `roleNames[]`.
2. `POST user_session {user_id}` → `token = session.token||session_token||session_id||id`.
3. `AuthResponse{token,user{…roles,person}}` → `TokenStorage.saveToken` (jwtDecode exp, `AsyncStorage auth_token`).
4. `getHighestRole(roles)` → `AsyncStorage userEmail` + tema/idioma por rol → `onLogin(role,token)`.
5. Refresh: `POST user_session/refresh {refresh_token}`. Logout: `DELETE user_session/{id}` (viewmodels hoy solo limpian local).
6. Recovery: `POST auth/forgot-password|verify-code|reset-password` (15s timeout, `VerificationError`, `MAX_CODE_ATTEMPTS=5`, lockout 30s, expiración 600s, resend 60s).

### Web (mock)
- Vía A (`AuthService`): dict `admin/teacher/student@example.com:123456` → JWT hardcodeado; `register` fabrica `eyJ…`.
- Vía B (`AuthContext`): `delay 800ms → mockAppUsers.find(email)`; `fetch("/api/auth/login")` comentado. Sesión en `localStorage @faceattend:session_user`.
- Sin refresh, sin `Bearer`, sin interceptores.

## 4. Endpoint usage

### Web: 0 HTTP de negocio
Todas las pantallas leen `useAppData()` / `mockData.ts` / `models/data/*Storage` (CRUD local). `FaceRegistrationModal` hace cropping local, no `POST face/*`.

### Mobile: servicios genéricos + N+1 en viewmodels
Patrón `getAll/getById/create/update/delete` sobre `app_user,person,user_session,academic_actor,attendance_record,justification,justification_type,course,program,school,academic_period,cohort,enrollment,environment,schedule_block,class_session,alert,academic_configuration,security_configuration,biometric_update_case,role,permission,city,audit_log,error_log,blood_type,document_type,language` + `face/register` multipart + `countriesnow` externo.

ViewModels con llamadas directas (`request()` / `fetch`):
- `useDashboardViewModel`: `GET attendance_record?_limit=100`, `GET class_session?class_session_id=`, `GET schedule_block?schedule_block_id=`, `GET course?course_id=`, `GET alert_type`, `GET alert?_sort=raised_at&_order=desc&_limit=5`, `GET academic_actor?academic_actor_id=`, `GET person?person_id=`, `GET justification?review_status=Pending`.
- `useDisplayingAttendanceViewModel`: `GET attendance_record?_limit=200` + enriquecimiento N+1 (actor→person, block→course→environment).
- `useAttendanceReportViewModel`: `GET attendance_record?_limit=500` + `POST attendance_report {role,attendance_type,people}` (endpoint no existe en backend — registrar como gap).
- `useAddJustifyViewModel`: `POST justification {attendance_record_id,justification_type_id,reason,submitted_at,review_status:Pending,attachment{…}}`.
- `useManageUsersViewModel`: `GET app_user?_limit=50` + `GET user_role?user_id=` + `GET person?person_id=`.
- `UpdatePhotoScreen`: `POST face/register` multipart.

## 5. DTOs, errores, paginación

- DTOs: `BaseModel{created_at,updated_at,deleted_at,created_by,updated_by,deleted_by,row_version}` + `fromApi/toApi` snake_case; `AuthRequest/AuthResponse`, `AttendanceRecord{Present/Absent/Late/Justified}`, roles `Administrador/Docente/Estudiante`.
- Errores: `ApiError(status,message,data)`; login mapea `0/408→connectionError`; viewmodels usan `useCustomAlert`; `LanguageService` fallback a `DEFAULT_LANGUAGES`; `CountryService` `{success,data/error}`.
- Paginación: solo `?_limit (+_sort/_order en alerts)`, sin `page/offset`. Migrar a `?limit&offset` del backend.

## 6. Migration checklist (Web + Mobile → Kong)

1. Apuntar `API_BASE_URL` a `http://localhost:8080` (env `EXPO_PUBLIC_API_URL` / Web `.env`).
2. Enviar `Authorization: Bearer` siempre (`requiresAuth:true`) + retry con refresh en 401.
3. Reemplazar `?_limit` por `?limit&offset` y leer `x-total-count` / `{data,total,limit,offset}`.
4. Eliminar N+1 cliente usando nested reads (`/class-sessions/{id}/attendance`, `/cohorts/{id}/blocks`, …).
5. Implementar `POST attendance_report` en backend o eliminar llamada Mobile.
6. Enviar `face/register` a `POST /api/v1/biometric/facial/enroll|verify` con JWT (no multipart anónimo).
