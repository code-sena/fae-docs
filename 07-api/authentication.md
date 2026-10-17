# Authentication & Authorization — FaceAttendEDU

> Decisions 2026-10-03 (updated 2026-10-07, verified against `FULL/`): **Kong authenticates at the
> edge.** A global `pre-function` (Lua) runs before routing on every non-public route: it requires
> `Authorization: Bearer <uuid>` and calls `GET ms-identity:8081/api/v1/sessions/{id}`, accepting
> only `sessionStatus = "Active"` (401 for a missing header, a non-UUID token or an
> invalid/expired/closed session; 503 when identity does not answer; **positive session checks are
> cached 10 s** at the edge via `kong.cache` — logout invalidates the entry, so the revocation
> window is 10 s; needs `KONG_UNTRUSTED_LUA_SANDBOX_REQUIRES=resty.http`). Kong still declares **no
> `jwt` plugin and no consumers**: the bearer is an **opaque session id, not a JWT**, and `jwt`
> RS256 issued by identity remains ADR-007's future evolution. The gateway authenticates at the
> edge, and **every one of the 9 services re-authenticates and evaluates RBAC again on its own**
> (updated 2026-10-11, re-verified against `FULL/`): not just the Java `AuthTokenFilter`
> (01/02/04/05) — `03`/`07`/`09` (TS) run `registerAuthGuard`, `06` (Python) runs
> `require_permission`, `08` (Go) runs `AuthMiddleware`. Same session+permission contract,
> duplicated per service on purpose (no shared module).

> Real strategy as implemented: sessions owned by `01-ms-identity`, RBAC owned by
> `02-ms-authorization`, edge session check in Kong (`pre-function`), and a second session+RBAC
> check inside every service (Java `AuthTokenFilter`, TS `registerAuthGuard`, Python
> `require_permission`, Go `AuthMiddleware`) — all four call the same two endpoints
> (`GET /sessions/{id}`, `GET /auth/evaluate`). Sources: `99-api-gateway/kong/kong.yml`,
> `01-ms-identity/.../AuthController|SessionController|AuthTokenFilter|SecurityConfig`,
> `02-ms-authorization/.../UserRoleController`, `03-ms-academic/src/infrastructure/http/
> authGuard.ts|permissions.ts`, `06-ms-biometric/infrastructure/web/security.py`,
> `07-ms-configuration/src/infrastructure/http/authGuard.ts|permissions.ts`,
> `08-ms-notification/internal/infrastructure/http/handler/auth_middleware.go`,
> `09-ms-quality/src/infrastructure/http/authGuard.ts`,
> `front-end/Mobile/src/services/AuthService.js`, `front-end/Web/src/context/AuthContext.js`.

## 1. Summary

- **Mechanism:** `Authorization: Bearer <sessionId>` where `sessionId` is an opaque UUID issued at
  login (never a JWT).
- **Edge enforcement (real):** Kong's global `pre-function` (Lua, `kong.yml`) runs before routing
  and calls `GET ms-identity:8081/api/v1/sessions/{id}`; it accepts only HTTP 200 with
  `sessionStatus = "Active"`. **401** = missing header, not `Bearer`, token that is not an
  8-4-4-4-12 UUID, or invalid/expired/closed session; **503** = identity does not answer
  (2 s/2 s/3 s timeouts). **Cache:** positive checks are cached **10 s** (`kong.cache`, keyed
  `faceattend:session:<id>`) and `/api/v1/auth/logout` invalidates the entry; every service guard
  keeps its own 10 s window, so revocation propagates within 10 s. Needs
  `KONG_UNTRUSTED_LUA_SANDBOX_REQUIRES=resty.http` on the gateway container.
- **Service enforcement (real), all 9 services:** `AuthTokenFilter` (Java 01/02/04/05, copy per
  MS) and its three equivalents — `registerAuthGuard` (TS 03/07/09, a Fastify `onRequest` hook),
  `require_permission` (Python 06, a FastAPI dependency), `AuthMiddleware` (Go 08, a Gin
  middleware) — all validate the bearer again on every `/api/**` call, load the session from
  `GET /api/v1/sessions/{id}` (identity), and evaluate a permission via
  `GET /api/v1/auth/evaluate` (authorization) whenever the route needs one. Java additionally
  loads roles from `GET /api/v1/users/{id}/roles` and propagates `X-User-Id` / `X-User-Roles`; the
  other three only set the resolved `userId` on the request. Public paths (session not required):
  `login`, `logout`, `refresh`, `forgot/verify/reset-password`, `me` (GET), `GET /sessions/*`,
  health/docs. **No shared module — each guard is a separate implementation, duplicated on
  purpose.** The gateway's edge check and every service's own check are both real; see
  `08-uml/diagrams/source/seq-login.mmd` (SEQ-02) for the full call chain.
- **No coverage gap today:** `03` academic, `06` biometric, `07` configuration, `08` notification
  and `09` quality (TS/Python/Go) each run the guard above — confirmed 2026-10-11 against `FULL/`
  (`03-ms-academic/src/infrastructure/http/authGuard.ts`,
  `06-ms-biometric/infrastructure/web/security.py`,
  `07-ms-configuration/src/infrastructure/http/authGuard.ts`,
  `08-ms-notification/internal/infrastructure/http/handler/auth_middleware.go`,
  `09-ms-quality/src/infrastructure/http/authGuard.ts`, all wired unconditionally at boot). A
  direct call to a service's own port (8083/8086/8087/8088/8089, published on loopback by
  compose) still has to pass that service's own session+permission check — it is **not** an open
  bypass. `15-project-control/risks.md` R-003 and `05-architecture/overview.md` §8 AT-004
  described an earlier state of the code and are now corrected to match.
- **Issuer (real):** `POST /api/v1/auth/login {identifier|email|username, password}` →
  `UserSessionDto{sessionId,userId,sessionStatus,startDate,endDate,sourceIp}`; failures always
  return **401 "Invalid email or password"** (no user enumeration).
- **Password hashing (real):** `BCryptPasswordEncoder($2B, strength 12)` in
  `01-ms-identity SecurityConfig`; checked in `AuthenticateUserUseCaseImpl`.
- **No secrets in the repo:** there is no HS256 consumer/secret in `kong.yml` any more (the old
  `faceattend-admin` consumer was removed with the jwt plugin).
- **RBAC:** `02-ms-authorization` owns `role,permission,role_permission,user_role`;
  evaluation endpoint `GET /api/v1/auth/evaluate?userId&permission → {userId,permission,allowed}`.

## 2. Gateway enforcement matrix (real `kong.yml`)

> The **global** plugins apply to every row below and run in this order: `pre-function` (edge
> session check, see §1) → `cors` (origins `["*"]`) → `rate-limiting
> 300/min` (redis). The third column lists the **per-upstream** plugins only — i.e. what Kong
> itself adds beyond the global pre-function. "None at Kong" never means unauthenticated: every
> upstream also runs its own session+RBAC guard (see §1), Kong is just not layering an extra
> plugin on top of the edge check for that route. No `jwt` plugin and no consumers anywhere.

| Upstream | Paths | Gateway plugins (per-upstream) |
|----------|-------|-----------------|
| identity `ms-identity:8081` | `/api/v1/persons,/users,/auth,/sessions,/password-policies` + nested `~/users/.+/activate` | `cors(*)` + `rate-limiting 60/min redis` — no `jwt` plugin |
| authorization `ms-authorization:8082` | `/api/v1/roles,/permissions` + `~/users/.+/roles`, `/api/v1/auth/evaluate` | `rate-limiting 100/min redis` — no `jwt` plugin |
| academic `ms-academic:8083` | `/api/v1/schools,/programs,/academic-periods,/periods,/cohorts,/courses,/actor-types,/academic-actors,/enrollments` | none at Kong — MS-side `registerAuthGuard` (TS) |
| scheduling `ms-scheduling:8084` | `/api/v1/environments,/schedule-blocks,/class-sessions` + nested `~/cohorts/.+/blocks` etc. | none at Kong — MS-side `AuthTokenFilter` (Java) |
| attendance `ms-attendance:8085` | `/api/v1/attendance-records,/justifications,/justification-types,/supporting-documents` + nested `~/class-sessions/.+/attendance` etc. | `rate-limiting 200/min redis` — no `jwt` plugin, MS-side `AuthTokenFilter` (Java) |
| biometric `ms-biometric:8086` | `/api/v1/biometric` | none at Kong — MS-side `require_permission` (Python); the MS returns raw embeddings if its own check is ever bypassed |
| configuration `ms-configuration:8087` | `/api/v1/academic-configurations,/security-configurations,/biometric-update-cases,/configurations` + nested | none at Kong — MS-side `registerAuthGuard` (TS) |
| notification `ms-notification:8088` | `/api/v1/alert-types,/alerts` | none at Kong — MS-side `AuthMiddleware` (Go) |
| quality-health | `/health/quality` → `ms-academic:8083` | `cors(*)` (only health route at the gateway; **no `GET /health`**) |
| quality `ms-quality:8089` | `/api/v1/quality` | `rate-limiting 100/min redis`, MS-side `registerAuthGuard` (TS) |
| global | all | `cors(origins ["*"])`, `rate-limiting 300/min redis` |

## 3. Identity flows (real)

### 3.1 Login / logout / me / refresh / recovery (`AuthController`, `PasswordRecoveryController`)

```
POST /api/v1/auth/login  {identifier|email|username, password}
      → 200 {sessionId,userId,sessionStatus,startDate,endDate,sourceIp}
      → 401 {"status":401,"error":"Unauthorized","message":"Invalid email or password"}
POST /api/v1/auth/refresh {sessionId}  → 200 UserSessionDto (new sessionId, old one closed)
      → 401 if the session expired or was already rotated (single use)
POST /api/v1/auth/logout?sessionId → 204
GET  /api/v1/auth/me?username|email  → UserDto   (one of the two is required → 400 otherwise)
POST /api/v1/auth/forgot-password {email}      → 202 {status:"accepted",message:"Request accepted"}
POST /api/v1/auth/verify-code {email,code}     → 200 {status:"verified",message:"Code verified"}
POST /api/v1/auth/reset-password {email,password} → 200 {status:"updated",message:"Password updated"}
POST /api/v1/sessions?userId&sourceIp → 201 UserSessionDto
GET  /api/v1/sessions?page&limit      → {data,meta{page,limit,total,totalPages}}
GET  /api/v1/sessions/{id}            → UserSessionDto (MS-public: used by every AuthTokenFilter
                                         and by the Kong pre-function — the edge still requires a bearer)
GET  /api/v1/sessions/user/{userId}   → [UserSessionDto]
DELETE /api/v1/sessions/{id}          → 204
```

- `AuthRequest{identifier}` with `@JsonAlias({"username","email"})` — clients may send
  `{"identifier":…}`, `{"email":…}` or `{"username":…}`.
- `UserDto{userId,personId,username,authenticationType:Local|Windows|External,status,lastAccess,createdAt,updatedAt}`,
  `PersonDto{personId,documentType,documentNumber,name,lastName,email,phone,bloodType,birthDate,address,…}`,
  `UserSessionDto{sessionId,userId,sessionStatus:Active|Closed,startDate,endDate,sourceIp}`.
- The session `sessionId` **is** the bearer token. `POST /api/v1/auth/refresh` rotates it: the
  body carries the current `sessionId`, the answer carries a new one and the previous row is
  closed. There is **no** separate refresh-token entity — the token is still an opaque session.
- Recovery is a 3-step, all-public flow: `forgot-password` answers **202 regardless of whether
  the account exists** (anti-enumeration) and emails a 6-digit code through the internal
  `POST ms-notification:8088/api/v1/emails` (no Kong route); the code is stored **hashed
  (SHA-256)** in memory with a **10 min TTL, 5 attempts and a 60 s resend cooldown**
  (`application.yml` → `auth.code-ttl-minutes|max-attempts|resend-cooldown-seconds`), all in
  constant-time comparison; `reset-password` consumes the verified challenge.

### 3.2 Users / persons / cities lifecycle

```
POST /api/v1/users · GET /api/v1/users?page&limit · GET/PUT /{id} (204) ·
     POST /{id}/activate (204) · PATCH /{id}/status?status (204)
POST /api/v1/persons · GET /api/v1/persons?page&limit · GET/PUT /{id} (204) ·
     PATCH /{id}/status (204)
POST /api/v1/persons/{id}/transfer — NOT IMPLEMENTED (person has no school_id; use academic_actor)
POST/GET/PUT/DELETE /api/v1/cities... — REMOVED (table dropped by Liquibase, no controller, no Kong route)
```

Lists are paged with **`?page&limit` (1-based page)** and return the `PageResponse`
`{data,meta{page,limit,total,totalPages}}` — see `pagination.md`.

## 4. RBAC model (`02-ms-authorization`)

Tables: `role{roleId,roleName,description}`, `permission{permissionId,permissionName,description}`,
`role_permission`, `user_role{userId→roleId}`.

```
GET/POST /api/v1/roles · GET/PUT/DELETE /api/v1/roles/{id}
POST /api/v1/roles/{roleId}/permissions {permissionId} · DELETE …/{permId} · GET …/permissions
GET/POST /api/v1/permissions · GET/PUT/DELETE /api/v1/permissions/{id}
POST /api/v1/users/{userId}/roles {roleId} · DELETE …/{roleId} · GET …/roles
GET /api/v1/user-roles?userIds → {userId: RoleResponse[]} (batch, max 300 ids; `{}` without ids)
GET /api/v1/auth/evaluate?userId&permission → {userId,permission,allowed}
```

Seed roles (changeset `02-ms-authorization-db/02-dml/004-unify-mobile-roles`):
**`Administrador | Instructor | Aprendiz`** (the older `SUPER_ADMIN/SCHOOL_ADMIN/INSTRUCTOR/STUDENT`
names are migrated away; clients still accept them as legacy aliases). 12 permissions
(`contexto.recurso:acción`, e.g. `attendance.record:read`) with per-role grants — see
`database/SEEDS.md`.

Kong itself does **not** call `/auth/evaluate` — the edge pre-function only checks that the
session is `Active`, never a permission. Every service's own guard does call it: Java
`AuthTokenFilter` evaluates the coarse `identity.person:read|write`; the TS/Python/Go guards
evaluate a route-derived permission of their own (e.g. `academic.school:manage`,
`attendance.record:write`, `biometric.case:request|review`, `configuration:manage`) — see
`03-ms-academic/src/infrastructure/http/permissions.ts`,
`07-ms-configuration/src/infrastructure/http/permissions.ts` and the equivalent files per
service. Clients also call `evaluate` directly for their own UI gating (Mobile calls
`evaluate(userId, 'attendance.record:read')` right after login) — that is a third, independent
check, not a substitute for the service-side one.

### Roles vs frontend

- Web: `core/utils/backendRoles.js` is the **single** role map
  (`Administrador|ADMIN→admin`, `Instructor|DOCENTE→teacher`, `Aprendiz|STUDENT|ESTUDIANTE→student`,
  plus legacy `SUPER_ADMIN/SCHOOL_ADMIN/RECTOR/COORDINATOR`); priority `admin > teacher > student`.
  The old `rolesHierarchy.ts` file no longer exists.
- Mobile: `GET api/v1/users/{id}/roles → roleName[]` then `getHighestRole()` with the same
  admin > teacher > student priority.

## 5. Frontend auth (real)

### 5.1 Mobile (`front-end/Mobile`, real backend)

- Base URL: `ENV.API_BASE_URL` = `http://<host>:8000/` (Kong) resolved from
  `EXPO_PUBLIC_API_URL` → Metro `hostUri` → `10.0.2.2` (Android emulator) → LAN default
  (`config/env.js`). All business traffic goes through the gateway; the direct MS ports **are**
  published on loopback (`127.0.0.1:8081-8089`, `BIND_IP`) for local dev — services 03/06/07/08/09
  have no auth filter of their own, so bypassing Kong means no authentication (`risks.md` R-003).
- Client: `api/apiClient.js` — `request({method,url,data,params,requiresAuth=true})`,
  `Authorization: Bearer <token>` attached automatically whenever a token exists,
  `API_TIMEOUT 15000`, `ApiError(status,message,data)`.
- Login (`services/AuthService.js`): `POST api/v1/auth/login {email,password}` (the backend
  resolves email → user via `identity.person.email`) → `saveToken(sessionId)` in
  `expo-secure-store` (AsyncStorage on web) → parallel `GET api/v1/users/{id}/roles` +
  profile (`GET users/{id}` / `persons/{id}`) → `GET api/v1/auth/evaluate?userId&permission=
  attendance.record:read` (fail-closed: no permission, no login) → `AuthResponse{token,user{…roles,person}}`.
- Logout: `POST api/v1/auth/logout?sessionId` (best-effort) then clear storage.
- Refresh: the backend implements `POST /api/v1/auth/refresh` (session rotation), but
  `AuthService.refreshToken()` still throws "Sesión no renovable: vuelve a iniciar sesión" —
  Mobile never calls it (client-side gap, not a backend gap).
- Recovery (`services/verificationService.js|passwordService.js`): `POST auth/forgot-password`,
  `POST auth/verify-code`, `POST auth/reset-password` — **all three are implemented in the
  backend** (`01-ms-identity`, verified 2026-10-07): 6-digit code, 10 min TTL, 5 attempts,
  60 s resend cooldown. Delivery depends on SMTP being configured (`ms-notification`).
- Face photo: `POST <API_BASE_URL>face/register` (multipart, no auth header) → `{embedding}`
  stored in `SecureStore face_<documento>`.

### 5.2 Web (`front-end/Web`, real backend since `c4fad72`)

- Base URL: `src/config/env.ts getApiBaseUrl()` = `EXPO_PUBLIC_API_URL` (build-time) or
  `http://localhost:8000`, overridable at runtime with `window.__FACEATTEND_API_URL__`;
  `API_MODE` is `live` unless `EXPO_PUBLIC_API_MOCK=1`.
- Client: `src/api/apiClient.js` — `request(path,{method,body,query})` with automatic
  `Authorization: Bearer <sessionId>`, page-envelope unwrapping, `fetchAllPages`, and
  `ApiError`; endpoints in `src/api/endpoints.js`.
- Services: `src/services/api/{auth,academic,authorization,configuration,notification,quality,
  scheduling,referenceData}Api.js` — all real HTTP.
- Login: `context/AuthContext.js` → `authApi.login({email,password})` →
  `POST /api/v1/auth/login`, saves the session with `saveToken(sessionId,{email})`
  (localStorage `@faceattend:session_user` on web) and loads roles/profile from the API.
- `mockData.js` survives only as fallback data when `API_MODE=mock`.
  `constants/urls.ts`, `services/AuthService.ts` and `rolesHierarchy.ts` were deleted.

## 6. Token lifetimes and rotation

| Token | Lifetime (real) | Rotation |
|-------|-----------------|----------|
| access = opaque `sessionId` (UUID) | until logout/expiry of the `user_session` row (`sessionStatus: Active`) | `POST /api/v1/auth/refresh` (single use): returns a new `sessionId` and closes the previous one; 401 if expired or already rotated. The Mobile client does not call it (§5.1) |
| recovery code | 600 s / 5 attempts / 60 s resend cooldown (`application.yml`, same values as the client `constants/auths.js`) | one code per challenge, hashed SHA-256 in memory; consumed by `POST /api/v1/auth/reset-password` |

## 7. Headers

- Frontend → Kong: `Authorization: Bearer <sessionId>`, `Content-Type: application/json`
  (or multipart for face).
- Java MS `AuthTokenFilter` → handler: sets request attributes `X-User-Id`, `X-User-Roles` and a
  response header `X-User-Roles` (comma-separated). TS/Go/Python services do not read them.
- CORS (global Kong plugin): origins `["*"]` (wildcard; was `localhost:3000/5173/8090` before —
  widened so LAN/phone clients work); methods GET/POST/PUT/PATCH/DELETE/OPTIONS; headers
  `Authorization,Content-Type,X-Requested-With`; `credentials:true`; `max_age:3600`.

## 8. Expiration, refresh, logout (normative)

1. On 401 (missing/invalid/expired session) → the client may try `POST /api/v1/auth/refresh`
   once (opaque session rotation); if that also answers 401, clear the stored session and
   redirect to login.
2. Re-login (or refresh) issues a brand-new `sessionId`; old sessions stay in `user_session`
   until closed.
3. Logout MUST call `POST /api/v1/auth/logout?sessionId` best-effort, then clear local storage
   even on failure.
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

Identity declares `security: []` at the root, so its contract shows no requirement — its own
`AuthTokenFilter` still enforces the bearer on every `/api/**` path except the bootstrap routes
(`/auth/login`, `/auth/logout`, `/auth/refresh`, `/auth/forgot-password`, `/auth/verify-code`,
`/auth/reset-password`, `GET /auth/me`, `GET /sessions/**`) and health/docs. At the edge only the
list in §1 is token-free, so `GET /sessions/{id}` needs a bearer when it is called **through
Kong** (identity accepts it without one so introspection works). The scheme is documented as an
opaque session id (not a JWT); `kong.yml` carries no consumer and no `jwt` plugin — the edge
check is the global `pre-function`.

## 10. Gaps and TODOs (honest)

- **Opaque session at the edge, not a signed token:** Kong authenticates by introspecting
  identity (cached 10 s per session, logout invalidates → +1 internal hop at most once per 10 s
  per session), so a stolen
  session id is valid until logout/expiry (revocation window = 10 s) and there is no revocation list besides the
  `user_session` row. Replacing the introspection with `jwt` RS256 + JWKS from identity
  (ADR-007, `AT-007`) is pending.
- **Duplicated auth code, not a missing check:** all 9 services independently re-implement the
  same session+RBAC contract (`03/07/09` TS `registerAuthGuard`, `06` Python `require_permission`,
  `08` Go `AuthMiddleware`, `01/02/04/05` Java `AuthTokenFilter`) because there is no shared
  module across stacks. This used to be a real gap (see git history) but is not one today —
  direct access to a service's published port still has to pass that service's own check.
  Internal ports should still stay inside the Docker network as defense in depth, but the risk is
  "one more layer to maintain, by hand, four times," not "no layer at all."
- **Coarse permission only (Java):** `AuthTokenFilter` checks `identity.person:read|write` only;
  the TS/Python/Go guards already do route-derived, finer-grained permissions
  (`academic.school:manage`, `attendance.record:write`, `biometric.case:request|review`, …) —
  bringing Java up to the same granularity, or wiring a shared Kong/OPA authorizer, is the
  remaining work.
- **Biometric update case has two unsynced implementations:** `ms-configuration`
  (`/api/v1/biometric-update-cases`, Postgres, canonical per the data model) and `ms-biometric`
  (`/api/v1/biometric/update-request*`, MongoDB) both implement "request + review a biometric
  update" independently — no HTTP call or event links them. A case created through one is
  invisible through the other, and the Mongo side skips the `fingerNumber`/`requestedBy`
  validation the Postgres side enforces. See `08-uml/diagrams/source/seq-biometric-update-
  duplication.mmd` (SEQ-04) and `state-biometric-update.mmd` (ST-03) for the full picture.
- **Recovery flow is implemented but unverified end to end:** the three routes
  (`auth/forgot-password|verify-code|reset-password`) exist in `01-ms-identity` (2026-10-07);
  delivery goes through `POST ms-notification:8088/api/v1/emails`, which answers
  `202 {status:"skipped"}` when `SMTP_HOST` is empty — verify the SMTP path before calling it
  done.
- **Password policy enforcement:** `password_policy` CRUD exists, but login/create-user flows do
  not yet validate against it (Mobile `PasswordService` only for reset).
- Clients must keep sending `Bearer` on every call; 401 handling (re-login) is client-side.
