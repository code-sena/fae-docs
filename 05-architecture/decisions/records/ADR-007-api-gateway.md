# ADR-007 — API Gateway Technology

- **ID:** ADR-007
- **Date:** 2026-09-12
- **Status:** Proposed
- **Implementation note (2026-10-07, verified against `FULL/`):** Kong OSS 3.6 runs **DB-less** with declarative `kong.yml`, routing by path to all 9 services, and **authenticates at the edge**: a global `pre-function` (Lua) requires `Authorization: Bearer <uuid>` on every non-public route and introspects `GET ms-identity:8081/api/v1/sessions/{id}`, accepting only `sessionStatus = "Active"` (401 for a missing/non-UUID/expired session, 503 when identity does not answer; no cache — one identity call per authenticated request; needs `KONG_UNTRUSTED_LUA_SANDBOX_REQUIRES=resty.http`). **The `jwt` plugin and its consumers are still not configured** — the other plugins are `cors` (origins `localhost:3000, 5173, 8090`) and `rate-limiting` (redis policy: global 300/min; identity 60, authorization 100, attendance 200, quality 100). The bearer is an **opaque session id, not a JWT**: `jwt` RS256 issued by identity remains the target state (`AT-007`). The gateway authenticates at the edge; permissions (RBAC) are then evaluated again by every service's own guard, not just Java — `AuthTokenFilter` (`01/02/04/05`), `registerAuthGuard` (`03/07/09`), `require_permission` (`06`), `AuthMiddleware` (`08`), re-verified 2026-10-11 against `FULL/`; details in `07-api/authentication.md`.
- **Authors:** FaceAttend EDU Team

---

## Context

The system exposes external clients (web, mobile, API) through a single edge component that handles routing, authentication, rate limiting, and CORS. The architecture overview (`05-architecture/overview.md` §7) and service catalog (`09-microservices/service-catalog.md`) already define the API Gateway as the entry point on port 8080.

The gateway must:
1. Route requests to the correct internal module (all 8 bounded contexts share a single deployable per ADR-002).
2. Validate the caller's credentials before forwarding (the decision below picks the token format; the current implementation validates an opaque session id, JWT RS256 is the target).
3. Apply rate limiting to prevent abuse (NFR-004).
4. Handle CORS for web clients.
5. Terminate TLS in production.
6. Be operationally simple for a 3-person team.

**Known constraints:**
- ADR-002 establishes a modular monolith — the gateway routes to a single application, not to 8 separate services (yet).
- The gateway must be ready for service extraction (ADR-002 Strangler Fig path).
- Team has no dedicated DevOps role.

---

## Decision

**We decided:** Use **Kong** (OSS) as the API Gateway, deployed as a Docker container alongside the application.

**Justification:**

1. **Mature plugin ecosystem:** Kong provides built-in plugins for JWT validation, rate limiting, CORS, IP restriction, and request transformation — no custom middleware needed.
2. **Declarative configuration:** Kong supports DB-less mode with `kong.yml` declarative config, eliminating the need for a separate database for gateway configuration.
3. **Extraction-ready:** When modules are extracted to separate services (ADR-002), Kong's service and route configuration simply adds new upstream targets — no gateway replacement needed.
4. **Industry standard:** Kong is one of the most widely deployed API gateways; extensive documentation and community support.
5. **Lightweight in DB-less mode:** Runs as a single NGINX-based container with no external database dependency for configuration.

**Implementation:**
- Deploy Kong in DB-less mode with declarative `kong.yml` configuration.
- A global `pre-function` (Lua) authenticates at the edge: it requires `Authorization: Bearer <uuid>` and validates the opaque session against identity (`GET /api/v1/sessions/{id}`, `sessionStatus = "Active"`). This is session introspection, not signature verification.
- Target: replace the introspection with the `jwt` plugin validating RS256 tokens issued by identity-service (public key / JWKS) — see *Consequences* and `AT-007`.
- Rate Limiting plugin enforces per-IP and per-user limits.
- CORS plugin handles cross-origin requests for web clients.
- The gateway authenticates at the edge; RBAC is then evaluated again by every service's own guard (Java `AuthTokenFilter`, TS `registerAuthGuard`, Python `require_permission`, Go `AuthMiddleware`), not just the Java ones.
- Kong routes to the microservice stack (ports 8081-8089 stay inside the Docker network; Kong listens on 8080).

---

## Evaluated alternatives

| Alternative | Pros | Cons | Reason for discarding |
|------------|------|------|-----------------------|
| **Kong OSS (chosen)** | Mature; plugin ecosystem; DB-less mode; declarative config; extraction-ready | Separate technology from Node.js services; Kong-specific YAML syntax; slightly heavier than NGINX-only | — (chosen) |
| NGINX + Lua (OpenResty) | Extremely fast; minimal resource usage; Lua scripting for custom logic | Requires Lua knowledge; no built-in JWT/rate-limiting plugins; more custom code to maintain | Less plugin ecosystem; more operational burden for custom logic |
| Express/Fastify custom gateway (Node.js) | Full TypeScript control; consistent with other services; custom middleware | Reinventing the wheel; worse performance than Kong; more maintenance burden; no built-in rate limiting or JWT validation | Not production-ready for an API gateway; excessive custom code |
| Traefik | Modern; auto-discovery with Docker; Let's Encrypt integration | Less mature plugin ecosystem; configuration model is different from Kong; smaller community for enterprise use cases | Kong has stronger plugin ecosystem and wider adoption |
| AWS API Gateway | Fully managed; zero operations; auto-scaling | Vendor lock-in (AWS); latency for external calls; cost per request; requires AWS account and networking setup | Team operates on-premise/local; vendor lock-in is undesirable |

---

## Consequences

**Positive:**
- Edge authentication, rate limiting, and CORS in one place — no custom middleware code in the services for the session check.
- DB-less mode reduces operational complexity (no separate gateway database).
- Declarative `kong.yml` is version-controlled and reviewable.
- Extraction-ready: adding new upstream services requires only config changes.

**Negative / Trade-offs:**
- Kong is a separate technology stack (Lua/NGINX-based) from the Node.js/Python services.
- Team must learn Kong's configuration syntax and plugin model.
- Kong adds a container to the Docker Compose stack.
- Session introspection on every request costs one internal HTTP call to identity (no cache) and adds latency; a signed `jwt` (RS256) would remove that hop.
- The gateway authenticates only: services without their own filter (03/06/07/08/09) are fully exposed if a request bypasses Kong (direct port access) — see `15-project-control/risks.md` R-003.

**Impact on the system:**
- Affected modules: all (all external traffic passes through Kong).
- Documents that must be updated:
  - `05-architecture/overview.md` §4 — fill in Kong as the gateway
  - `09-microservices/service-catalog.md` — confirm Kong on port 8080
  - `10-devops/local-setup.md` — add Kong container to Docker Compose
  - `00-governance/security-policy.md` — reference Kong's edge session validation (and the JWT target state)

---

## Risks

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|-----------|
| Kong configuration complexity grows with routes/plugins | Medium | Low | Start with minimal config (edge session check, rate limit, CORS); document each route in `kong.yml` with comments |
| Kong becomes a single point of failure | Low | Medium | Run Kong as a Docker container with restart policy; health check monitoring; can add replica in production |
| DB-less mode limits some advanced features (analytics, ACL) | Low | Low | DB-less mode covers all current requirements; can migrate to DB-backed mode if needed later |

---

## References

- Architecture overview → `05-architecture/overview.md` §4, §7
- Service catalog → `09-microservices/service-catalog.md`
- Security policy → `00-governance/security-policy.md`
- Related to: ADR-002 (extraction path requires gateway reconfiguration)
