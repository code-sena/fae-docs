# 07 — API Contracts

> **What is this?** The formal specification of how microservices communicate with each other
> and with the outside world. The contract is the promise a service makes to its consumers.
> All contracts here mirror REAL code in `../FACEATTEND-EDU     Front-end   Web/back-end`,
> `99-api-gateway/kong/kong.yml` and `front-end/{Web,Mobile}`.

## Why contracts are critical in microservices

In a monolith, changing a function is easy: the compiler tells you what you break.
In microservices, a change to an API can silently break other services in production.

**Contract-first development:** design the API before implementing it. This way the team
can work in parallel (frontend, backend, other services) with an agreed contract.

---

## Map (real services, ports, contracts)

| # | Service | Port | Stack | Contract | Auth at Kong |
|---|---------|------|-------|----------|--------------|
| 01 | Identity | 8081 | Java Spring | `contracts/openapi/01-identity-service.yaml` | public (cors + 60/min) |
| 02 | Authorization | 8083 | Java Spring | `contracts/openapi/02-authorization-service.yaml` | jwt + 100/min |
| 03 | Academic | 8084 | TS Fastify | `contracts/openapi/03-academic-service.yaml` | jwt |
| 04 | Scheduling | 8087 | Java Spring | `contracts/openapi/04-scheduling-service.yaml` | jwt |
| 05 | Attendance | 8085 | Java Spring | `contracts/openapi/05-attendance-service.yaml` | jwt + 200/min |
| 06 | Biometric | 8086 | Python FastAPI | `contracts/openapi/06-biometric-service.yaml` | jwt |
| 07 | Configuration | 8089 | TS Fastify | `contracts/openapi/07-configuration-service.yaml` | jwt |
| 08 | Notification | 8090 | Go Gin | `contracts/openapi/08-notification-service.yaml` | jwt |
| 09 | Quality | 8091 | TS Fastify | `contracts/openapi/09-quality-service.yaml` | jwt + 100/min |

> Decisions 2026-10-03: Quality 09 is tooling owned by `11-quality`, not a DDD bounded context (00-06 stay at 8 contexts). Port matrix is frozen from `FULL/back-end/docker-compose.yml` + `99-api-gateway/kong/kong.yml` as canonical; this table is the published view.
| 99 | Gateway | 8080/8001 | Kong + Redis | `contracts/openapi/api-gateway.yaml` | global cors + 300/min |

Shared: `contracts/openapi/_shared.yaml`. Template: `_template-service.yaml`. Lint: `redocly.yaml`.

---

## Documents

### `guidelines.md` ⭐
REST standards grounded in real code: versioning `/api/v1`, kebab-case plurals, POST/PATCH actions
(`/open|close|cancel`, `/review`, `/activate`, `/transfer`, `/resolve`), nested reads, audit fields,
health/quality probes, contract-first checklist. **Read before designing any endpoint.**

### `authentication.md` ⭐
Kong JWT at the edge (HS256 dev `faceattend-admin-key`, RS256 prod target), public identity vs jwt rest,
RBAC (`/auth/evaluate`), Mobile real multistep login vs Web mock, refresh/logout, header conventions, gaps.

### `versioning.md`
URL-only `/api/v1`, breaking vs non-breaking, deprecation policy, current `1.0.0`.

### `pagination.md`
`?limit=20&offset=0` canonical; Java lists filter in memory; TS `x-total-count` (+`{data,total,limit,offset}` in 09);
Go `?actorId&typeId&status&limit&offset` with `{alerts}/{alert_types}`; Python none; Mobile `?_limit` legacy to migrate.

### `error-catalog.md`
Canonical `ErrorResponse` + per-stack wire formats (Java, Java ISO-9001, TS, Python 422, Go `{error:string}`,
Mobile `ApiError`), frontend mapping, rules.

### `endpoint-matrix.md` ⭐
Canonical METHOD×path table for all 9 services + gateway (Kong coverage, auth, description).

### `frontend-integration.md`
Web (mock-only, zero HTTP, migration path) vs Mobile (real `apiClient`, services, N+1 viewmodels, face multipart).

### `examples.md`
Copy-paste `curl` via `:8080` for login, each domain, health.

### `contracts/openapi/` ⭐⭐
One `.yaml` per microservice (OpenAPI 3.0.3), each with `servers` (direct + Kong + dev/prod),
`operationId`, `tags`, request/response schemas, `400/401/403/404/409/422/500`, reuse of `_shared.yaml`.

### `_shared.yaml`
`ErrorResponse`, `PaginatedMeta`, `LimitOffsetMeta`, `UUID/Timestamp/Date/Time`, `AuditFields`,
`HealthResponse`, `QualityReport`, `Page/Limit/Offset/Id/Correlation` params, `TotalCount` header,
`BadRequest/Unauthorized/Forbidden/NotFound/Conflict/Unprocessable/InternalError`, `bearerAuth`.

---

## Recommended tools

- **Swagger Editor** — Online editor for OpenAPI
- **Redocly** — Generates HTML documentation from the YAML (configured in `redocly.yaml`; run `npx @redocly/cli lint`)
- **Postman / Insomnia** — Import `contracts/openapi/*.yaml` + use `examples.md`
- **OpenAPI Generator** — Generates client/server code from the YAML

---

## Correlations with other sections

| This section is fed by... | And feeds into... |
|---------------------------|-------------------|
| `04-requirements/functional.md` → what it must do | `endpoint-matrix.md` maps each FR to endpoints |
| `06-data/domains/*.md` + `../database/*` → tables/columns | Schemas in each contract mirror DDL |
| `05-architecture/*`, `09-microservices/*` → hexagonal ports | HTTP adapters mirror ports |
| `back-end/0*` + `kong/kong.yml` → real routes | Contracts here (verified, not aspirational) |
| `front-end/Mobile|Web` → consumption | `frontend-integration.md` + migration checklist |
| Contracts here | `09-microservices/[service]/` implementation |
| Contracts here | `11-quality/testing-strategy.md` → contract tests |

---

## Questions this section answers

- What is each endpoint called and what does it do? → `endpoint-matrix.md` + per-service yaml.
- What data does it receive and what does it return? → request/response schemas per operation.
- How does the client authenticate? → `authentication.md` + `api-gateway.yaml`.
- What errors can the service return? → `error-catalog.md` + per-operation responses.
- How do I version the API without breaking consumers? → `versioning.md` + `guidelines.md §2`.
- How do Web/Mobile integrate? → `frontend-integration.md` + `examples.md`.
