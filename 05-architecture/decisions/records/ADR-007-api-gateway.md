# ADR-007 — API Gateway Technology

- **ID:** ADR-007
- **Date:** 2026-09-12
- **Status:** Proposed
- **Implementation note (2026-10-05, verified against `FULL/`):** Kong OSS 3.6 runs **DB-less** with declarative `kong.yml`, routing by path to all 9 services. **The `jwt` plugin and its consumers are not configured** — only `cors` (origins `localhost:3000, 5173, 8090`) and `rate-limiting` (redis policy: global 300/min; identity 60, authorization 100, attendance 200, quality 100). Authentication therefore works as opaque `sessionId` Bearer validated per service (`07-api/authentication.md`); edge JWT remains the target state for this ADR (`AT-007`).
- **Authors:** FaceAttend EDU Team

---

## Context

The system exposes external clients (web, mobile, API) through a single edge component that handles routing, authentication, rate limiting, and CORS. The architecture overview (`05-architecture/overview.md` §7) and service catalog (`09-microservices/service-catalog.md`) already define the API Gateway as the entry point on port 8080.

The gateway must:
1. Route requests to the correct internal module (all 8 bounded contexts share a single deployable per ADR-002).
2. Validate JWT tokens and enforce authentication before forwarding.
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
- JWT plugin validates tokens issued by identity-service (RS256 public key).
- Rate Limiting plugin enforces per-IP and per-user limits.
- CORS plugin handles cross-origin requests for web clients.
- Kong routes to the single modular monolith application (port 8081-8089 are internal; Kong listens on 8080).

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
- Built-in JWT validation, rate limiting, and CORS eliminate custom middleware code.
- DB-less mode reduces operational complexity (no separate gateway database).
- Declarative `kong.yml` is version-controlled and reviewable.
- Extraction-ready: adding new upstream services requires only config changes.

**Negative / Trade-offs:**
- Kong is a separate technology stack (Lua/NGINX-based) from the Node.js/Python services.
- Team must learn Kong's configuration syntax and plugin model.
- Kong adds a container to the Docker Compose stack.

**Impact on the system:**
- Affected modules: all (all external traffic passes through Kong).
- Documents that must be updated:
  - `05-architecture/overview.md` §4 — fill in Kong as the gateway
  - `09-microservices/service-catalog.md` — confirm Kong on port 8080
  - `10-devops/local-setup.md` — add Kong container to Docker Compose
  - `00-governance/security-policy.md` — reference Kong JWT plugin for token validation

---

## Risks

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|-----------|
| Kong configuration complexity grows with routes/plugins | Medium | Low | Start with minimal config (JWT, rate limit, CORS); document each route in `kong.yml` with comments |
| Kong becomes a single point of failure | Low | Medium | Run Kong as a Docker container with restart policy; health check monitoring; can add replica in production |
| DB-less mode limits some advanced features (analytics, ACL) | Low | Low | DB-less mode covers all current requirements; can migrate to DB-backed mode if needed later |

---

## References

- Architecture overview → `05-architecture/overview.md` §4, §7
- Service catalog → `09-microservices/service-catalog.md`
- Security policy → `00-governance/security-policy.md`
- Related to: ADR-002 (extraction path requires gateway reconfiguration)
