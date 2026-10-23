# Deployment Environments — FaceAttend EDU

> Defines the system's environments, their purposes, access, and key configurations.
> **Rule:** If you need to access production directly, something is wrong in the process.
> Most problems should be detected in staging.
>
> **Status note (updated 2026-10-17):** the project runs a fully containerized
> local stack via Docker Compose and a **production deployment on AWS EC2**
> (t3.xlarge, us-east-1, 13 containers, Neon PostgreSQL — deployed 2026-10-16,
> see `FULL/DESPLIEGUE.md` and ADR-020). The **web SPA publishes on Vercel**.
> **CI/CD runs on GitHub Actions** (`.github/workflows/`): `ci.yml` tests every
> push/PR to `main`, and `deploy-production.yml` deploys on every push to `main`
> through the chain database (Neon) → backend (EC2) → frontend (Vercel), gated
> by the manual approval of the `production` GitHub environment (details in
> [`ci-cd.md`](./ci-cd.md)). dev/staging environments are
> planned but not operational. This document describes what exists today and the
> target model for the remaining environments.

---

## Project environments

| Environment | Purpose | Update | Access | Data |
|-------------|---------|--------|--------|------|
| **local** | Development on the dev's machine (full Docker Compose stack: Postgres 17 + 8 Liquibase migrations, MongoDB 7, Redis 7, Kafka 3.8 KRaft, 9 microservices, Kong gateway, web frontend) | Manual (`docker compose up -d --build`) | Individual dev | Synthetic / seed (`tools/seed-api-test-data`, `database/database-init`) |
| **dev** | Continuous integration, shared team validation | Automatic (each PR merge) — **CI only, deploy planned** | Entire team | Synthetic |
| **staging** | Pre-production, QA, demos (IEEE 829 regression runs, PO demos) | Automatic (each merge to `main`) — **planned** | Team + PO | Anonymized from prod (once prod exists); for now synthetic |
| **production** | Real users (institutions: attendance via biometric recognition) | `deploy-production.yml` on push to `main` (database → backend → frontend), each step gated by manual approval of the `production` environment | Tech Lead + DevOps only | Real (Neon PostgreSQL) |

### What exists today vs. what is planned

| Item | Local | Dev | Staging | Production |
|---|---|---|---|---|
| Docker Compose stack (root `docker-compose.yml` incl. `database/` + `back-end/`) | ✅ running | ⚠️ same compose, hosted runner — planned | ❌ planned | ✅ running on AWS EC2 (ADR-020) |
| Secrets | `.env` from `.env.example` (dev values only, gitignored) | env injection (Vault/K8s Secrets) — planned | planned | `~/.env.prod` on the EC2 host (secrets manager pending) |
| DB migrations | ✅ Liquibase chain runs automatically on `up` (identity → authorization → academic → scheduling → attendance → biometric → configuration → notification) | same chain | same chain | same chain against Neon |
| Health verification | ✅ `healthcheck` per service + `docker compose ps` | smoke tests — planned | smoke + k6 perf — planned | healthchecks + Portainer; canary monitoring planned |

---

## Environment variables per environment

> Specific variables go in the environment's secrets system (Vault, AWS Secrets, K8s Secrets).
> Local values go in `.env.example` (without real data).
>
> **Project convention:** there is a **single `.env` at the repo root**
> (`FULL/.env`, copied from `.env.example`). Docker Compose loads it once and
> interpolates every included file (`database/`, `back-end/`). Services running
> outside Docker read the *same* file at startup: Java via
> `spring.config.import optional:file:...`, Node (academic, quality) via
> `process.loadEnvFile('../../.env')`, Python (biometric) via `_load_root_env()`
> in `settings.py`. All compose entries have safe defaults
> (`${POSTGRES_USER:-postgres}`), so **any run works without a `.env`**.
> Never commit real secrets; override them via shell environment or a secrets
> manager in deployed environments.

### Naming convention

The project follows framework-native naming (Spring relaxed binding, Kong `KONG_*`,
Docker image envs) plus a project prefix `FACEATTEND_*` for cross-service URLs
and internal tokens:

```
Framework standard:   SPRING_DATASOURCE_URL, SERVER_PORT, NODE_ENV, KONG_DATABASE
Data stores (images): POSTGRES_USER, MONGO_INITDB_ROOT_USERNAME, KAFKA_*
Project-specific:     FACEATTEND_[TARGET]_[VARIABLE]

Examples:
  FACEATTEND_AUTH_IDENTITY_URL          # ms-* -> identity session validation
  FACEATTEND_AUTH_AUTHORIZATION_URL     # ms-* -> RBAC evaluate
  FACEATTEND_NOTIFICATION_BASE_URL      # identity -> email channel
  FACEATTEND_NOTIFICATION_INTERNAL_TOKEN# shared secret (X-Internal-Token)
Infrastructure/host:  BIND_IP, FRONTEND_PORT, EXPO_PUBLIC_API_URL, SEED_*, BOOTSTRAP_*
```

### Variables per environment (example values — real ones in the vault)

| Variable | Local (`.env.example`) | Dev | Staging | Production |
|----------|------------------------|-----|---------|------------|
| `POSTGRES_USER` / `POSTGRES_PASSWORD` / `POSTGRES_DB` | `postgres` / `postgres` / `faceattend_db` | Vault | Vault | Vault |
| `MONGO_USER` / `MONGO_PASSWORD` / `MONGO_DB` | `mongoadmin` / `mongopass` / `faceattend_biometric` | Vault | Vault | Vault |
| `DATABASE_URL` (TS services: academic, configuration, notification, quality) | interpolated from `POSTGRES_*` → `postgres://…@postgres:5432/faceattend_db` | Vault | Vault | Vault |
| `SPRING_DATASOURCE_URL/_USERNAME/_PASSWORD` (Java: identity, authorization, scheduling, attendance) | interpolated from `POSTGRES_*` (JDBC, one schema each via `SPRING_DATASOURCE_HIKARI_SCHEMA`) | Vault | Vault | Vault |
| `MONGODB_URL` / `SPRING_DATA_MONGODB_URI` (biometric, attendance) | interpolated from `MONGO_*` | Vault | Vault | Vault |
| `KAFKA_BROKERS` / `SPRING_KAFKA_BOOTSTRAP_SERVERS` | `kafka:29092` (container network) | Vault | Vault | Vault |
| `EMAIL_INTERNAL_TOKEN` / `FACEATTEND_NOTIFICATION_INTERNAL_TOKEN` | empty = no check (dev); any GUID locally | Vault | Vault | Vault |
| `SMTP_HOST` / `SMTP_PORT` / `SMTP_USERNAME` / `SMTP_PASSWORD` / `SMTP_FROM` / `SMTP_STARTTLS` | MailHog/Mailpit: `host.docker.internal:1025`, STARTTLS `false`; empty `SMTP_HOST` disables email channel (202 skipped) | Vault | Vault (real account) | Vault (real account) |
| `FACEATTEND_LIQUIBASE_IMAGE` | `liquibase/liquibase:4.29.0` | pinned tag | pinned tag | pinned tag |
| `BIND_IP` | `127.0.0.1` (loopback; `0.0.0.0` only to debug from LAN/phone) | n/a (internal) | n/a | n/a |
| `KONG_PROXY_PORT` / `KONG_ADMIN_PORT` | `8000` / `8001` (admin always loopback-bound) | `80/443` behind TLS | `80/443` behind TLS | `80/443` behind TLS, **admin API never exposed** |
| `API_DOMAIN` (ADR-022) | not used (optional `DOMAIN` for the `docker-compose.tls.yml` overlay; defaults to `localhost` with Caddy's local CA) | required by `docker-compose.prod.yml`: DNS name pointing at the host, used by `Caddyfile.prod` for the Let's Encrypt certificate | same | same |
| `FRONTEND_PORT` | `8090` | 80/443 | 80/443 | 80/443 |
| `EXPO_PUBLIC_API_URL` (baked at frontend build time) | `http://localhost:8000` (physical devices: PC LAN IP, e.g. `http://192.168.1.10:8000`) | `https://dev.[domain]` | `https://staging.[domain]` | `https://[domain]` |
| `SEED_USERNAME` / `SEED_PASSWORD`, `BOOTSTRAP_USERNAME` / `BOOTSTRAP_PASSWORD` | dev-only test accounts (`SeedAdmin123!`, `Admin123!ChangeMe`) | synthetic seed | **not present** | **never** |
| `LOG_LEVEL` | Spring default (`info`; debug per-package when needed) | `info` | `info` | `warn` |
| `NODE_ENV` | `development` (outside Docker) / `production` (inside containers for ms-academic, ms-configuration, ms-quality) | `production` | `production` | `production` |
| `CORS_ORIGIN` (Kong global plugin, `kong.yml`) | `["*"]` — wildcard, any origin accepted (was `localhost:3000/5173/8090` before; widened so LAN/phone clients work) | `["*"]` | `["*"]` | lock down to `https://[domain]` |

> **Auth note:** the platform uses **opaque server-side sessions** — a row in
> `identity.user_session` (PostgreSQL) validated by Kong's `pre-function` and a
> 10 s positive cache — **not JWT, and not Redis** (Redis is used only for Kong
> rate-limiting). There is **no `JWT_SECRET` variable**. When RS256
> JWT is reintroduced (ADR-007), provision keys through the Kong Admin API —
> Kong cannot interpolate env vars in declarative config, so never commit them
> in `kong.yml`.
>
> **Per-service ports** are fixed by compose: `SERVER_PORT`/`PORT` = 8081–8089
> (identity 8081, authorization 8082, academic 8083, scheduling 8084,
> attendance 8085, biometric 8086, configuration 8087, notification 8088,
> quality 8089). Not environment-dependent; do not parameterize them.

---

## CI/CD Pipeline

> **Current state (verified in `.github/workflows/`):** **CI** (`ci.yml`, 157
> lines) tests every push/PR to `main` with per-stack matrices (Java
> `mvn test`, Node `npm test` + build for 03/07/08/09 and both frontends incl.
> `expo export`, Python `pytest`). **Deploy exists**: `deploy-production.yml`
> runs on every push to `main` (or manual dispatch of `database`/`backend`/
> `frontend`) through the chain **CI → database (Neon, Liquibase) → backend
> (EC2, SSH) → frontend (Vercel)**, with each step gated by the manual
> approval of the `production` GitHub environment. Dependabot is configured
> but paused (`open-pull-requests-limit: 0`). Details in [`ci-cd.md`](./ci-cd.md).

```
Developer
  │
  │ git push → PR against `main`
  │
  ▼
┌─────────────────────────────────────────────────────────┐
│  CI (ci.yml — every push/PR to main)                    │
│  ─────────────────────────────────────────────          │
│  1. Build deps + install   (< 2 min)                    │
│     - npm ci (Node services, Web, Mobile)               │
│     - mvn compile (Java) · poetry install (Python)      │
│  2. Unit Tests               (< 5 min)                  │
│     - mvn test (01,02,04,05) · npm test (03,07,08,09)   │
│     - pytest (06) · npm test (Web, Mobile)              │
│  3. Build + audit            (< 3 min)                  │
│     - npm run build (Node services)                     │
│     - expo export --platform web (Web)                  │
│     - npm audit / bandit (informational, non-blocking)  │
│  ─────────────────────────────────────────────          │
│  Note: there is no separate lint/format gate in CI today │
│  (eslint / tsc --noEmit / ruff run locally, not in the  │
│  workflow) — adding one is open tech debt.               │
│  If a job fails: blocks the PR merge                    │
└─────────────────────────────────────────────────────────┘
  │
  │ PR approved and merged to `main`
  │
  ▼
┌─────────────────────────────────────────────────────────┐
│  DEPLOY PRODUCTION (deploy-production.yml — push main)  │
│  ─────────────────────────────────────────────          │
│  0. Re-runs CI (deploy only proceeds if green)          │
│  1. Database (Neon)            [approval gate]          │
│     - Liquibase via docker-compose.neon.yml (update)    │
│  2. Backend (EC2)              [approval gate]          │
│     - SSH + deploy-backend-remote.sh                    │
│     - docker compose -f docker-compose.prod.yml --build │
│     - health wait (≤10 min) + Kong check :8000          │
│  3. Frontend (Vercel)          [approval gate]          │
│     - remote build with Vercel env vars                 │
│  Failure at any step stops the chain.                   │
│  Rollback: manual (redeploy previous SHA on EC2).       │
└─────────────────────────────────────────────────────────┘

Target (not operational yet): dev/staging environments — automated deploy on
merge with the seed smoke test (`tools/seed-api-test-data`), contract and k6
performance suites, and a canary split of production traffic once an
orchestrator supports it. Canary/Blue-Green remain targets: with a single
Compose stack there is no traffic splitting.

---

## Production deploy strategy

### Available options

| Strategy | Description | Risk | Rollback |
|----------|-------------|------|---------|
| **Canary** | 10% → 50% → 100% with validation at each step | Low | Automatic |
| **Blue-Green** | Instant switch between two versions | Medium | Immediate |
| **Rolling** | Replaces instances one by one | Low | Slow |
| **Recreate** | Takes everything down and starts the new version | High | Manual |

**Adopted strategy:** **Rolling** per service with Docker Compose
(`docker compose up -d` recreates only changed services; every `ms-*` exposes
`/health` or `/actuator/health` and Kong waits for `service_healthy`).
Canary is the target once an orchestrator with traffic splitting (K8s +
Ingress/Gateway API, or Kong upstream weights) exists — with a single compose
stack there is no way to split traffic by percentage. **Recreate is forbidden**:
the Liquibase chain already guarantees additive, backward-compatible
migrations, so downtime deploys are unnecessary.

### Production deploy checklist

- [ ] The release PR was reviewed and approved
- [ ] Staging tests passed (including performance)
- [ ] The PO approved the HUs included in this release
- [ ] The Tech Lead approved the deploy
- [ ] Someone is available to monitor the first 30 min post-deploy
- [ ] Database changes: the Liquibase chain ran **before** the code deploy
      (`cd database && docker compose up liquibase` order is enforced by the
      root compose dependency chain: identity → authorization → academic →
      scheduling → attendance → biometric → configuration → notification)
- [ ] No `SPRING_JPA_HIBERNATE_DDL_AUTO=update` leaks to prod (dev-only safety
      net; prod must rely exclusively on Liquibase — set `validate`/`none`)
- [ ] `kong.yml` CORS origins updated to the real domain; Kong Admin port (8001)
      not exposed publicly
- [ ] SMTP credentials configured (email channel active) and `EMAIL_INTERNAL_TOKEN`
      set to a random secret
- [ ] Frontend rebuilt with the correct `EXPO_PUBLIC_API_URL` (baked at build time)
- [ ] The rollback runbook is available: `[link to runbook]`

### Rollback

```bash
# Per-service rollback with Docker Compose (redeploy previous image tag)
docker compose pull [service]        # if registry-pinned tags are used
docker compose up -d --no-deps [service]   # recreate only that service

# Example: roll back only ms-attendance
docker compose up -d --no-deps ms-attendance

# Verify health after rollback (all services expose health endpoints)
docker compose ps                    # STATUS must show (healthy)
curl -s http://localhost:8000/api/v1/auth/... # smoke via Kong gateway

# Full stack stop WITHOUT deleting volumes (data preserved)
docker compose down

# Kubernetes variant (when adopted)
kubectl rollout undo deployment/[service-name] -n [namespace]
kubectl rollout status deployment/[service-name] -n [namespace]

# If there is an incompatible DB migration (EMERGENCY — consult Tech Lead first)
cd database/0X-ms-[context]-db
docker compose run --rm liquibase rollback-count 1
# NOTE: prefer forward-fix (new changeset) over rollback; Liquibase rollback
# requires a <rollback> block in each changeset. NEVER run `down -v`:
# it destroys postgres_data / mongo_data / kafka_data / redis_data.
```

---

## Deploy log

| Date | Service | Version | Environment | Deployed by | Issues |
|------|---------|---------|-------------|-------------|--------|
| 2026-10-16 | Full backend (9 ms + Kong + web) | compose `latest` | production (AWS EC2 t3.xlarge, Neon PostgreSQL) | Team | None documented — see `FULL/DESPLIEGUE.md` |

*(Deploy record started 2026-10-16; the first production deployment is documented in `FULL/DESPLIEGUE.md`.)*

---

## Correlations

- Local setup & full infra reference → `COMPOSE.md` (repo root) + `.env.example`
- Microservice map, stacks & ports → `back-end/SERVICES.md`
- Database design, schemas & seeds → `back-end/DATABASE.md`, `database/ESTRUCTURA.md`, `database/SEEDS.md`, `database/CONVENCIONES.md`
- Gateway routes, CORS & rate-limiting → `back-end/99-api-gateway/kong/kong.yml`, `back-end/99-api-gateway/SERVICE.md`
- Test data seeding (dev/staging smoke) → `tools/seed-api-test-data/README.md`
- Quality reports (ISO 25010 / ISTQB / IEEE 829) → `back-end/QUALITY.md` and `back-end/ISO25010_QUALITY_REPORT.md`, `back-end/ISTQB_QUALITY_REPORT.md`, `back-end/IEEE829_TEST_PLAN.md`
- Post-deploy observability → `13-operations/observability.md` *(planned; current observability = `docker compose logs -f [service]` + Kong access log on stdout)*
- Runbook per service → `09-microservices/services/XX/runbook.md` *(planned; use each service's `SERVICE.md` meanwhile)*
- Branch policy → `00-governance/git-conventions.md` *(planned)*