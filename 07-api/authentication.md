# Authentication & Authorization — FaceAttendEDU

> Decisions 2026-10-03 (updated 2026-10-07, verified against `FULL/`): **Kong authenticates at the
> edge.** A global `pre-function` (Lua) runs before routing on every non-public route: it requires
> `Authorization: Bearer <uuid>` and calls `GET ms-identity:8081/api/v1/sessions/{id}`, accepting
> only `sessionStatus = "Active"` (401 for a missing header, a non-UUID token or an
> invalid/expired/closed session; 503 when identity does not answer; no cache — one identity call
> per request; needs `KONG_UNTRUSTED_LUA_SANDBOX_REQUIRES=resty.http`). Kong still declares **no
> `jwt` plugin and no consumers**: the bearer is an **opaque session id, not a JWT**, and `jwt`
> RS256 issued by identity remains ADR-007's future evolution. The gateway **only authenticates**
> — permissions (RBAC) are still evaluated per service by `AuthTokenFilter` in the Java services
> (01/02/04/05).

> Real strategy as implemented: sessions owned by `01-ms-identity`, RBAC owned by
> `02-ms-authorization`, edge session check in Kong (`pre-function`), permissions in the Java MSs.
> Sources: `99-api-gateway/kong/kong.yml`, `01-ms-identity/.../AuthController|SessionController|
> AuthTokenFilter|SecurityConfig`, `02-ms-authorization/.../UserRoleController`,
> `front-end/Mobile/src/services/AuthService.js`, `front-end/Web/src/context/AuthContext.js`.

## 1. Summary

- **Mechanism:** `Authorization: Bearer <sessionId>` where `sessionId` is an opaque UUID issued at
  login (never a JWT).
- **Edge enforcement (real):** Kong's global `pre-function` (Lua, `kong.yml`) runs before routing
  and calls `GET ms-identity:8081/api/v1/sessions/{id}`; it accepts only HTTP 200 with
  `sessionStatus = "Active"`. **401** = missing header, not `Bearer`, token that is not an
  8-4-4-4-12 UUID, or invalid/expired/closed session; **503** = identity does not answer
  (2 s/2 s/3 s timeouts). **No cache**: every authenticated request costs one call to identity.
  Needs `KONG_UNTRUSTED_LUA_SANDBOX_REQUIRES=resty.http` on the gateway container.
- **Service enforcement (real):** `AuthTokenFilter` (copy per MS, no shared module) validates the
  bearer again on every `/api/**` call, loads the session from `GET /api/v1/sessions/{id}`
  (identity), loads roles from `GET /api/v1/users/{id}/roles` (authorization), evaluates the
  coarse permission (`identity.person:read|write` in identity) via `GET /api/v1/auth/evaluate`,
  then propagates `X-User-Id` / `X-User-Roles`. Its public paths: `login`, `logout`, `me` (GET),
  `GET /sessions/*`, health/docs. **The gateway authenticates only — RBAC lives here.**
- **Coverage gap:** only `01/02/04/05` (Java) carry the filter. `03` academic, `06` biometric,
  `07` configuration, `08` notification and `09` quality (TS/Python/Go) have **no auth check of
  their own** — Kong validates the session for them at the edge (`gateway only` in
  `endpoint-matrix.md`), but a direct call to their ports (8083, 8086, 8087, 8088, 8089 — all
  published on loopback by compose) or from any container on the compose network bypasses every
  check, and `ms-biometric` returns raw facial/fingerprint embeddings. Known risk:
  `15-project-control/risks.md` R-003, `05-architecture/overview.md` §8 AT-004. Internal ports
  must never be exposed outside the Docker network.
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
> session check, see §1) → `cors` (origins `localhost:3000, 5173, 8090`) → `rate-limiting
> 300/min` (redis). The third column lists the **per-upstream** plugins only. No `jwt` plugin and
> no consumers anywhere.

| Upstream | Paths | Gateway plugins (per-upstream) |
|----------|-------|-----------------|
| identity `ms-identity:8081` | `/api/v1/persons,/users,/auth,/sessions,/cities,/password-policies` + nested `~/users/.+/activate` | `cors(*)` + `rate-limiting 60/min redis` — no `jwt` plugin |
| authorization `ms-authorization:8082` | `/api/v1/roles,/permissions` + `~/users/.+/roles`, `/api/v1/auth/evaluate` | `rate-limiting 100/min redis` — no `jwt` plugin |
| academic `ms-academic:8083` | `/api/v1/schools,/programs,/academic-periods,/periods,/cohorts,/courses,/actor-types,/academic-actors,/enrollments` | none (gateway-only auth) |
| scheduling `ms-scheduling:8084` | `/api/v1/environments,/schedule-blocks,/class-sessions` + nested `~/cohorts/.+/blocks` etc. | none — MS-side `AuthTokenFilter` |
| attendance `ms-attendance:8085` | `/api/v1/attendance-records,/justifications,/justification-types,/supporting-documents` + nested `~/class-sessions/.+/attendance` etc. | `rate-limiting 200/min redis` — no `jwt` plugin |
| biometric `ms-biometric:8086` | `/api/v1/biometric` | none (gateway-only auth — the MS returns raw embeddings) |
| configuration `ms-configuration:8087` | `/api/v1/academic-configurations,/security-configurations,/biometric-update-cases,/configurations` + nested | none (gateway-only auth) |
| notification `ms-notification:8088` | `/api/v1/alert-types,/alerts` | none (gateway-only auth) |
| quality-health | `/health/quality` → `ms-academic:8083` | `cors(*)` (only health route at the gateway; **no `GET /health`**) |
| quality `ms-quality:8089` | `/api/v1/quality` | `rate-limiting 100/min redis` (gateway-only auth) |
| global | all | `cors(origins localhost:3000, 5173, 8090)`, `rate-limiting 300/min redis` |

## 3. Identity flows (real)

### 3.1 Login / logout / me (`01-ms-identity/.../AuthController.java`)

```
POST /api/v1/auth/login  {identifier|email|username, password}
      → 200 {sessionId,userId,sessionStatus,startDate,endDate,sourceIp}
      → 401 {"status":401,"error":"Unauthorized","message":"Invalid email or password"}
POST /api/v1/auth/logout?sessionId → 204
GET  /api/v1/auth/me?username|email  → UserDto   (one of the two is required → 400 otherwise)
POST /api/v1/sessions?userId&sourceIp → 201 UserSessionDto
GET  /api/v1/sessions?page&limit      → {data,meta{page,limit,total,totalPages}}
GET  /api/v1/sessions/{id}            → UserSessionDto (public: used by every AuthTokenFilter)
GET  /api/v1/sessions/user/{userId}   → [UserSessionDto]
DELETE /api/v1/sessions/{id}          → 204
```

- `AuthRequest{identifier}` with `@JsonAlias({"username","email"})` — clients may send
  `{"identifier":…}`, `{"email":…}` or `{"username":…}`.
- `UserDto{userId,personId,username,authenticationType:Local|Windows|External,status,lastAccess,createdAt,updatedAt}`,
  `PersonDto{personId,documentType,documentNumber,name,lastName,email,phone,bloodType,birthDate,address,…}`,
  `UserSessionDto{sessionId,userId,sessionStatus:Active|Closed,startDate,endDate,sourceIp}`.
- The session `sessionId` **is** the bearer token; there is no refresh/rotation endpoint.

### 3.2 Users / persons / cities lifecycle

```
POST /api/v1/users · GET /api/v1/users?page&limit · GET/PUT /{id} (204) ·
     POST /{id}/activate (204) · PATCH /{id}/status?status (204)
POST /api/v1/persons · GET /api/v1/persons?page&limit · GET/PUT /{id} (204) ·
     PATCH /{id}/status (204)
POST /api/v1/persons/{id}/transfer — NOT IMPLEMENTED (person has no school_id; use academic_actor)
POST/GET/PUT/DELETE /api/v1/cities… — legacy: `identity.city` is deprecated in the model
     (cities come from the external `countriesnow.space` catalog) but the endpoints and the
     Liquibase table are still served.
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
GET /api/v1/auth/evaluate?userId&permission → {userId,permission,allowed}
```

Seed roles (changeset `02-ms-authorization-db/02-dml/004-unify-mobile-roles`):
**`Administrador | Instructor | Aprendiz`** (the older `SUPER_ADMIN/SCHOOL_ADMIN/INSTRUCTOR/STUDENT`
names are migrated away; clients still accept them as legacy aliases). 12 permissions
(`contexto.recurso:acción`, e.g. `attendance.record:read`) with per-role grants — see
`database/SEEDS.md`.

Kong does **not** call `/auth/evaluate`; fine-grained checks live only inside the Java
`AuthTokenFilter` (coarse `identity.person:read|write`) and in the clients (Mobile calls
`evaluate(userId, 'attendance.record:read')` right after login). Wire evaluate into the
non-Java services — tech-debt, §10.

### Roles vs frontend

- Web: `core/utils/backendRoles.js` is the **single** role map
  (`Administrador|ADMIN→admin`, `Instructor|DOCENTE→teacher`, `Aprendiz|STUDENT|ESTUDIANTE→student`,
  plus legacy `SUPER_ADMIN/SCHOOL_ADMIN/RECTOR/COORDINATOR`); priority `admin > teacher > student`.
  The old `rolesHierarchy.ts` file no longer exists.
- Mobile: `GET api/v1/users/{id}/roles → roleName[]` then `getHighestRole()` with the same
  admin > teacher > student priority.

## 5. Frontend auth (real)

### 5.1 Mobile (`front-end/Mobile`, real backend)

- Base URL: `ENV.API_BASE_URL` = `http://<host>:8080/` (Kong) resolved from
  `EXPO_PUBLIC_API_URL` → Metro `hostUri` → `10.0.2.2` (Android emulator) → LAN default
  (`config/env.js`). All business traffic goes through the gateway; direct MS ports are not
  published.
- Client: `api/apiClient.js` — `request({method,url,data,params,requiresAuth=true})`,
  `Authorization: Bearer <token>` attached automatically whenever a token exists,
  `API_TIMEOUT 15000`, `ApiError(status,message,data)`.
- Login (`services/AuthService.js`): `POST api/v1/auth/login {email,password}` (the backend
  resolves email → user via `identity.person.email`) → `saveToken(sessionId)` in
  `expo-secure-store` (AsyncStorage on web) → parallel `GET api/v1/users/{id}/roles` +
  profile (`GET users/{id}` / `persons/{id}`) → `GET api/v1/auth/evaluate?userId&permission=
  attendance.record:read` (fail-closed: no permission, no login) → `AuthResponse{token,user{…roles,person}}`.
- Logout: `POST api/v1/auth/logout?sessionId` (best-effort) then clear storage.
- Refresh: **none** — `AuthService.refreshToken()` throws "Sesión no renovable: vuelve a iniciar sesión".
- Recovery (`services/verificationService.js|passwordService.js`): `POST auth/forgot-password`,
  `POST auth/verify-code`, `POST auth/reset-password` — **no such routes exist in the backend
  yet** (404 today; client contract retained, see §10).
- Face photo: `POST <API_BASE_URL>face/register` (multipart, no auth header) → `{embedding}`
  stored in `SecureStore face_<documento>`.

### 5.2 Web (`front-end/Web`, real backend since `c4fad72`)

- Base URL: `src/config/env.ts getApiBaseUrl()` = `EXPO_PUBLIC_API_URL` (build-time) or
  `http://localhost:8080`, overridable at runtime with `window.__FACEATTEND_API_URL__`;
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
| access = opaque `sessionId` (UUID) | until logout/expiry of the `user_session` row (`sessionStatus: Active`) | **none** — no refresh endpoint |
| recovery code | client contract 600s / 5 attempts (`constants/auths.js`) | backend routes not implemented |

## 7. Headers

- Frontend → Kong: `Authorization: Bearer <sessionId>`, `Content-Type: application/json`
  (or multipart for face).
- Java MS `AuthTokenFilter` → handler: sets request attributes `X-User-Id`, `X-User-Roles` and a
  response header `X-User-Roles` (comma-separated). TS/Go/Python services do not read them.
- CORS (global Kong plugin): origins `http://localhost:3000`, `http://localhost:5173`,
  `http://localhost:8090`; methods GET/POST/PUT/PATCH/DELETE/OPTIONS; headers
  `Authorization,Content-Type,X-Requested-With`; `credentials:true`; `max_age:3600`.

## 8. Expiration, refresh, logout (normative)

1. On 401 (missing/invalid/expired session) → clear the stored session and redirect to login;
   there is **no** refresh endpoint to try.
2. Re-login issues a brand-new `sessionId`; old sessions stay in `user_session` until closed.
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

Identity marks the bootstrap routes (`/auth/login`, `/auth/logout`, `/auth/me`,
`GET /sessions/{id}`, `GET /sessions/user/{userId}`) as `security: []`. The scheme is documented
as an opaque session id (not a JWT); `kong.yml` carries no consumer and no jwt plugin.

## 10. Gaps and TODOs (honest)

- **Opaque session at the edge, not a signed token:** Kong authenticates by introspecting
  identity on every authenticated request (no cache → +1 internal hop per call), so a stolen
  session id is valid until logout/expiry and there is no revocation list besides the
  `user_session` row. Replacing the introspection with `jwt` RS256 + JWKS from identity
  (ADR-007, `AT-007`) is pending.
- **TS/Python/Go services are open:** `03/06/07/08/09` accept any request that reaches them.
  Kong authenticates them at the edge, but direct access to their published ports (or from any
  container on the compose network) bypasses the gateway entirely — biometric even serves raw
  embeddings. Internal ports must stay inside the Docker network (`risks.md` R-003).
- **Coarse permission only:** `AuthTokenFilter` checks `identity.person:read|write`; a full
  per-route permission map (or Kong/OPA authorizer wired to `/auth/evaluate`) is missing.
- **Recovery endpoints missing:** Mobile calls `auth/forgot-password|verify-code|reset-password`
  which the backend does not implement.
- **Password policy enforcement:** `password_policy` CRUD exists, but login/create-user flows do
  not yet validate against it (Mobile `PasswordService` only for reset).
- Clients must keep sending `Bearer` on every call; 401 handling (re-login) is client-side.
