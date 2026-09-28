# Authentication & Authorization — FaceAttendEDU

> Real strategy as implemented: Kong JWT at the edge, permissive services behind it,
> RBAC owned by `02-ms-authorization`, sessions owned by `01-ms-identity`.
> Sources: `99-api-gateway/kong/kong.yml`, `01-ms-identity/.../AuthController|SessionController|SecurityConfig`,
> `02-ms-authorization/.../UserRoleController`, `front-end/Mobile/src/services/AuthService.js`,
> `front-end/Web/src/services/AuthService.ts|context/AuthContext.tsx`.

## 1. Summary

- **Mechanism:** JWT Bearer (`Authorization: Bearer <token>`). Gateway validates; microservices trust gateway headers.
- **Issuer (real):** `01-ms-identity` (`POST /api/v1/auth/login` → `UserSessionDto{sessionId,userId,sessionStatus}`; `POST /api/v1/sessions?userId&sourceIp`).
- **Dev secret (real):** Kong consumer `faceattend-admin`, `HS256`, key `faceattend-admin-key`, secret `faceattend-local-dev-secret-change-me` (`kong.yml:145-152`). **Rotate before prod.**
- **Prod target (declared):** RS256 keys issued by identity-service (`kong.yml` comment). Not yet enforced in code.
- **Local services:** `SecurityConfig/AppConfig` set `permitAll` for `/api/v1/**` (all Java services); TS/Python/Go have no local JWT check. Auth is **edge-only** today.
- **RBAC:** `02-ms-authorization` owns `role,permission,role_permission,user_role`; evaluation endpoint `GET /api/v1/auth/evaluate?userId&permission → {userId,permission,allowed}`.

## 2. Gateway enforcement matrix (real `kong.yml`)

| Upstream | Paths | Gateway plugins |
|----------|-------|-----------------|
| identity `ms-identity:8081` | `/api/v1/persons,/users,/auth,/sessions,/cities,/password-policies` + nested `~/persons/.+/transfer`, `~/users/.+/activate` | `cors(*)`, `rate-limiting 60/min redis` — **no `jwt` (public)** |
| authorization `ms-authorization:8083` | `/api/v1/roles,/permissions` + `~/users/.+/roles`, `/api/v1/auth/evaluate` | `jwt` + `rate-limiting 100/min` |
| academic `ms-academic:8084` | `/api/v1/schools,/programs,/academic-periods,/periods,/cohorts,/courses,/actor-types,/academic-actors,/enrollments` | `jwt` |
| scheduling `ms-scheduling:8087` | `/api/v1/environments,/schedule-blocks,/class-sessions` + `~/cohorts/.+/blocks` | `jwt` |
| attendance `ms-attendance:8085` | `/api/v1/attendance-records,/justifications,/justification-types,/supporting-documents` + nested `~/class-sessions/.+/attendance` etc. | `jwt` + `rate-limiting 200/min` |
| biometric `ms-biometric:8086` | `/api/v1/biometric` | `jwt` |
| configuration `ms-configuration:8089` | `/api/v1/academic-configurations,/security-configurations,/biometric-update-cases,/configurations` + nested | `jwt` |
| notification `ms-notification:8090` | `/api/v1/alert-types,/alerts` | `jwt` |
| quality `ms-quality:8091` | `/api/v1/quality` | `jwt` + `rate-limiting 100/min` |
| global | all | `cors(localhost:3000,5173)`, `rate-limiting 300/min redis` |

## 3. Identity flows (real)

### 3.1 Login / logout / me (`01-ms-identity/.../AuthController.java`)

```
POST /api/v1/auth/login            {username,password} → {sessionId,userId,sessionStatus}
POST /api/v1/auth/logout?sessionId → 204
GET  /api/v1/auth/me?username      → {username}
POST /api/v1/sessions?userId&sourceIp → {sessionId,userId,sessionStatus}
DELETE /api/v1/sessions/{id}      → 204
GET /api/v1/sessions/user/{userId} → [{sessionId,userId,sessionStatus}]
```

- No password hashing/refresh documented in code; session token IS the bearer token in current Mobile flow.
- `UserDto{userId,username,authenticationType,status}`, `PersonDto{personId,name,lastName,email}`, `UserSessionDto{sessionId,userId,sessionStatus}`.

### 3.2 Users / persons lifecycle

```
POST /api/v1/users / GET /{id} / PUT /{id} (204) / POST /{id}/activate (204) / PATCH /{id}/status?status (204)
POST /api/v1/persons / GET /{id} / PUT /{id} (204) / POST /{id}/transfer {newSchoolId} (204) / PATCH /{id}/status (204)
```

## 4. RBAC model (`02-ms-authorization`)

Tables: `role{roleId,roleName,description}`, `permission{permissionId,permissionName,description}`, `role_permission`, `user_role{userId→roleId}`.

```
GET/POST /api/v1/roles, GET/PUT/DELETE /api/v1/roles/{id}
POST /api/v1/roles/{roleId}/permissions {permissionId} / DELETE …/{permId} / GET …/permissions
GET/POST /api/v1/permissions, GET/PUT/DELETE /api/v1/permissions/{id}
POST /api/v1/users/{userId}/roles {roleId} / DELETE …/{roleId} / GET …/roles
GET /api/v1/auth/evaluate?userId&permission → {userId,permission,allowed}
```

Seed roles (per `SERVICE.md`): `SUPER_ADMIN,SCHOOL_ADMIN,INSTRUCTOR,STUDENT`.
Gateway SHOULD call `/auth/evaluate` for fine-grained checks; today only coarse `jwt` is enforced — evaluation is available but not wired as a Kong authorizer. Record as tech-debt.

### Roles vs frontend

- Web `ROLES_HIERARCHY=[Administrator,Teacher,Student]` (`rolesHierarchy.ts`), `getHighestRole()` picks max privilege. Note: legacy code spells these values in Spanish (`Administrador,Docente,Estudiante`).
- Mobile resolves `GET user_role?user_id → loop GET role?role_id → roleNames[]` then `getHighestRole()`.

## 5. Frontend auth (real)

### 5.1 Mobile (`front-end/Mobile`, consumes backend)

- Base URL: `ENV.API_BASE_URL` from `EXPO_PUBLIC_API_URL` or `host:3000` autodetect, fallback `http://10.3.234.223:3000/` (`config/env.js`).
- Client: `api/apiClient.js` (`fetch`, `TIMEOUT 15000`, `ApiError(status,message,data)`, `buildHeaders` adds `Bearer` only if `requiresAuth=true` — today all callers pass `false`, so header is never sent; auth relies on Dev open network / JSON-Server).
- Login (`services/AuthService.js:26-94`): `GET person?email → GET app_user?person_id → GET user_role?user_id → loop GET role?role_id → POST user_session {user_id} → token=session.token||session_token||session_id||id → AuthResponse{token,user{…roles,person}} → TokenStorage.saveToken (jwtDecode exp)`.
- Refresh: `POST user_session/refresh {refresh_token}`; logout: `DELETE user_session/{sessionId}` (viewmodels currently only clear local storage).
- Recovery: `POST auth/forgot-password {email}`, `POST auth/verify-code {email,code}`, `POST auth/reset-password {email,password}` with 15s timeout + `VerificationError{INVALID_CODE,EXPIRED_CODE,NO_CODE,TIMEOUT,GENERIC}`.
- Face photo: `POST <API_BASE_URL>face/register` (`multipart photo+documento+nombre`, no auth header) → `{embedding}` stored in `SecureStore face_<documento>`.

### 5.2 Web (`front-end/Web`, mock-only today)

- No real HTTP for business entities. `api/apiClient.ts` is a stub comment; `constants/urls.ts` (`http://10.3.234.12:8080/` + paths) has zero imports.
- `services/AuthService.ts`: mock dict `admin/teacher/student@example.com:123456` → hardcoded JWT; `register` fabricates `eyJ…` token.
- `context/AuthContext.tsx`: mock `delay 800ms → mockAppUsers.find(email)`; real `fetch("/api/auth/login")` is commented out (lines 106-114). Session in `localStorage @faceattend:session_user` (web) / `AsyncStorage` (native).
- Migration path: point Web `apiClient` at Kong `:8080`, reuse Mobile `AuthService.login` sequence, then send `Authorization: Bearer` on every call and honor 401 → refresh → retry.

## 6. Token lifetimes and rotation

| Token | Lifetime (documented intent) | Rotation |
|-------|------------------------------|----------|
| access JWT (HS256 dev) | 1h (target; today session-bound, no exp enforced at gateway) | Re-login / `POST user_session/refresh` (mobile) |
| refresh/session id | 7d intent (today opaque `sessionId`) | Single-use rotation (target; not enforced) |
| recovery code | 600s, max 5 attempts, 30s lockout, 60s resend (`constants/auths.js`) | Backend-owned |

## 7. Headers

- Frontend → Kong: `Authorization: Bearer <token>`, `Content-Type: application/json` (or multipart for face).
- Kong → upstream (convention to adopt, partially present): `X-User-Id`, `X-User-Role`, `X-Correlation-Id` / `X-Request-Id`. Python already emits `x-request-id,x-process-time-ms`; Go/TS/Java should propagate `traceId/correlationId` (see `error-catalog.md`).
- CORS: `Authorization,Content-Type,X-Requested-With`; `credentials:true`; `max_age:3600`.

## 8. Expiration, refresh, logout (normative)

1. On 401 from Kong → client MUST try refresh once (`POST /api/v1/auth/refresh` or `POST user_session/refresh`) then retry original request once.
2. On second 401 → clear `auth_token` + `userRole/userEmail` and redirect to login.
3. Logout MUST call `POST /api/v1/auth/logout?sessionId` (or `DELETE user_session/{id}`) best-effort, then clear local storage even on failure.
4. Never store passwords; never log tokens; never accept `?token=` query auth.

## 9. OpenAPI security

All contracts except identity declare:

```yaml
security:
  - bearerAuth: []
components:
  securitySchemes:
    bearerAuth:
      $ref: './_shared.yaml#/components/securitySchemes/bearerAuth'
```

Identity marks `/auth/*`, `/users`, `/persons`, `/cities`, `/sessions`, `/password-policies` as `security: []` (public at gateway) with a note that prod MAY add `jwt` selectively.

## 10. Gaps and TODOs (honest)

- HS256 dev secret in repo; needs RS256 rotation + JWKS endpoint (`GET /auth/keys` in `auth-service.yaml` is aspirational, not implemented).
- No per-role `@PreAuthorize`; only gateway `jwt`. Wire `GET /auth/evaluate` into Kong or service middleware.
- Mobile `requiresAuth:false` everywhere; Web has no client. Both must send Bearer in prod.
- Password validation in Mobile login is email-existence only; backend must enforce password check + lockout (`ACTIVE,INACTIVE,LOCKED` already in User model).
