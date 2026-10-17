# 10 — DevOps

> **What is this?** How code goes from the developer's computer to production,
> and how it is maintained in each environment. CI/CD, environments, and local configuration.

## Why this section exists

Without documented DevOps:
- Each developer has their own deploy process (and fails differently)
- "It works on my machine" is the most expensive answer in software
- Releases are high-risk events instead of routine

**FaceAttend EDU context:** this project is a polyglot monorepo (Java/Spring,
TypeScript/Fastify, Python/FastAPI, Go/Gin + two Expo frontends) orchestrated
end-to-end with Docker Compose. The single source of truth for "how everything
runs" is the root `docker-compose.yml`, which includes `database/docker-compose.yml`
(Postgres 17 + Liquibase chain + MongoDB + Redis + Kafka) and `back-end/docker-compose.yml`
(9 microservices + Kong gateway + web frontend). DevOps documentation here must
always reflect that compose model — if a document contradicts `COMPOSE.md`, the
compose files win and the document is a bug.

---

## What is here and how to fill it in

### `local-setup.md` ⭐ (Most urgent) — ✅ written → [`local-setup.md`](./local-setup.md)

How to start THE ENTIRE SYSTEM on a new computer from scratch.
**Filled with:** step by step, omitting nothing, assuming the reader has never touched the project.
Includes: software prerequisites (Docker path as golden path + per-service toolchains),
environment variables (root `.env` with safe defaults — stack works even without `.env`),
exact commands (`docker compose up -d --build` from the root), automatic Liquibase
migration chain, verification via Kong smoke route + login + seed tool, common issues
(Kong waiting for healthy upstreams, port clashes, `EXPO_PUBLIC_API_URL` baked at build time).

**Format used (adapted to reality):**
```markdown
## Prerequisites
- [x] Docker Desktop >= 24.0 (+ Compose >= 2.20)
- [x] Git >= 2.40
- [x] Node.js >= 20 LTS (frontends, TS services, seed tool)
- [x] JDK 21 + Maven wrapper (only for native Java dev)
- [x] Python 3.12 + Poetry (only for ms-biometric native dev)
- [x] Go >= 1.22 (only for ms-notification native dev)

## Steps
1. Clone the repository: `git clone <org>/FaceAttend-Edu.git && git checkout develop`
2. Copy environment variables: `cp .env.example .env`   (optional — defaults are built in)
3. Start the whole stack: `docker compose up -d --build`   (from repo root)
4. Wait for readiness: Liquibase chain runs automatically; then `docker compose ps`
5. Verify: `curl -s http://localhost:8000/health/quality` + login through Kong
6. Optional E2E seed/smoke: `cd tools/seed-api-test-data && npm run seed`

## Local ports
| Service | Port |
|---------|------|
| API Gateway (Kong proxy) | 8000 |
| Web SPA (nginx) | 8090 |
| Kong Admin (loopback only) | 8001 |
| ms-identity / authorization / academic / scheduling / attendance | 8081–8085 |
| ms-biometric / configuration / notification / quality | 8086–8089 |
| PostgreSQL 17 (8 schemas) | 5432 |
| MongoDB 7 | 27017 |
| Redis 7 | 6379 |
| Kafka 3.8 KRaft | 9092 |
| Mailpit UI (optional, not in compose) | 8025 |

## Common issues
- **Port in use:** `lsof -i :8000` (frontend must be 8090, NOT 8082 — taken by ms-authorization)
- **DB not connecting:** verify Docker is running; containers talk via network `faceattend-data` using host `postgres:5432`
- **Kong stuck `starting`:** it waits for all 8 ms-* healthy — find the failing one with `docker compose logs ms-X`
```

### `environments.md` ⭐ — ✅ written → see [`environments.md`](./environments.md)

Description of all project environments.
**Filled with:** local, development, staging, production. For each: purpose, URL,
who has access, how it is deployed, what data it has. Plus variable conventions
(framework-native naming + `FACEATTEND_*` prefix — there is **no `JWT_SECRET`**:
auth uses opaque sessions in Redis validated via Kong), CI/CD target pipeline,
production strategy (**Rolling with Docker Compose today; Canary once an
orchestrator exists**), rollback runbook, deploy log.

**Format used (adapted):**
```markdown
| Environment | URL | Purpose | Access | DB | Deploy |
|-------------|-----|---------|--------|-----|--------|
| local | localhost:8000 (Kong) / :8090 (web) | Individual development | All | Postgres+Mongo synthetic seeds | Manual (`docker compose up -d --build`) |
| dev | dev.[domain] — planned | Continuous integration | Team | Synthetic | Automatic on merge to `develop` — planned |
| staging | staging.[domain] — planned | QA / validation / PO demos | Team + PO | Anonymized (later) | Automatic on merge to `main` — planned |
| prod | api.[domain] — planned | Real users | Tech Lead + DevOps only | Real | Manual approval, Rolling deploy |
```

### `ci-cd.md` ⭐ — ⚠️ target state for **deploy**; **CI already exists**

Description of the continuous integration and deployment pipeline.
**Current truth:** **CI runs on GitHub Actions** — `.github/workflows/ci.yml` (171 lines) tests
every push/PR to `main` with per-stack matrices (Java `mvn test`, Node `npm test` + build,
Python `pytest`, Go `go test`, both frontends incl. `expo export`), and `.github/dependabot.yml`
gives weekly update PRs for all eight ecosystems. **CD (deploy) is still manual**: production is
deployed over SSH with `docker compose up -d --build` on the AWS EC2 host (see ADR-018/ADR-020
and `environments.md`). The pipeline below remains the target for deploy automation.

**Target format:**
```markdown
## PR Pipeline (runs on every Pull Request) — planned tool: GitHub Actions
1. Lint & format: eslint (frontends) · tsc --noEmit (TS services) · mvn compile (Java) · ruff (Python) · go vet (Go)
2. Unit tests: ./mvnw test (01,02,04,05) · npm test (03,07,09) · poetry run pytest (06) · make test (08)
3. Integration tests: supertest/testcontainers suites (ms-academic) against ephemeral Postgres
4. Coverage analysis (minimum threshold to be agreed with Tech Lead)
5. Security analysis (SAST: dependabot/CodeQL + docker image scan)
6. Build: docker compose config (validate model) + per-service Dockerfile builds
→ If any step fails: PR merge is blocked

## Pipeline on merge to `develop` (deploy to dev)
1. All of the above
2. Docker image build (faceattend/ms-*:latest + frontend-web)
3. Deploy to dev environment (compose up -d on dev host)
4. Smoke tests in dev: tools/seed-api-test-data (exit 0 = pass) + GET health via Kong

## Release pipeline to production
1. Merge develop → main triggers staging deploy + full suite + k6 performance (biometric match, attendance marking)
2. Acceptance test suite executed by QA (IEEE 829 regression plan)
3. PO approves the HUs included in the release
4. Manual approval by Tech Lead
5. Deploy to production: Rolling per service (compose recreate; canary only after K8s adoption)
6. Post-deploy monitoring 30 min; rollback = redeploy previous tag (see environments.md § Rollback)
```

---

## Correlations with other sections

| This section is fed by... | And feeds into... |
|---------------------------|-------------------|
| `COMPOSE.md` + root `docker-compose.yml` → how everything runs locally | Pipeline steps (what CI must build/run) |
| `back-end/IEEE829_TEST_PLAN.md`, `ISTQB_QUALITY_REPORT.md` → which tests to run | Pipeline gates (PR/dev/staging) |
| `back-end/SERVICES.md` + each `back-end/0X-ms-*/Dockerfile` → services to deploy | Which images the pipeline builds (9 ms-* + frontend-web) |
| `database/CONVENCIONES.md` + Liquibase chain → migration discipline | Deploy checklist (migrations before code) |
| `13-operations/` → post-deploy monitoring *(planned; today = `docker compose logs` + Kong access log)* | What the pipeline verifies (smoke/health) |

Related files inside this section:
- [`local-setup.md`](./local-setup.md) — complete local setup guide (golden path)
- [`environments.md`](./environments.md) — environments, variables, deploy strategy, rollback
- `ci-cd.md` — will live here once pipeline files are added to the repo (target defined below)

---

## Questions this section must answer

- **How do I start the system locally in 30 minutes?**
  `git clone` → `cp .env.example .env` (optional) → `docker compose up -d --build` →
  `docker compose ps` all `(healthy)` → `curl http://localhost:8000/health/quality`.
  Full walkthrough in [`local-setup.md`](./local-setup.md). First build ~15–30 min
  (image pulls + 4 Maven builds); subsequent starts are minutes.

- **What happens automatically when I push?**
  **CI on GitHub Actions**: every push/PR to `main` runs the per-stack test matrices
  (Java/Node/Python/Go + both frontends) and blocks merges on failure; Dependabot
  opens weekly update PRs. **Deploy is still manual** (no CD yet) — see `ci-cd.md`.

- **How does code get to production?**
  `feature/HU-XXX` → PR → `develop` → release PR `develop → main` (staging + full
  suite + perf) → PO sign-off → Tech Lead manual approval → Rolling deploy per
  service with health-gated recreation; migrations run before code (Liquibase
  chain); rollback = redeploy previous tag or `liquibase rollback-count 1`
  (forward-fix preferred). Detailed in [`environments.md`](./environments.md).

- **How many environments are there and what is each one for?**
  Four defined: **local** (dev machine, Docker Compose, synthetic/seed data),
  **dev** (CI target), **staging** (QA/demos), **production** (real users, manual
  approval only). **Local and production exist operationally** — production runs on
  AWS EC2 with Neon PostgreSQL, and the web SPA publishes on Vercel (see
  `environments.md` and ADR-020). dev/staging are still specified and planned.