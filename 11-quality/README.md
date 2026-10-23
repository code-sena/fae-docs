# 11 — Quality · Testing Strategy (FaceAttend EDU)

> Comprehensive testing strategy for the project. Sources in the repository:
> `back-end/IEEE829_TEST_PLAN.md`, `back-end/IEEE829_TEST_CASE_SPECIFICATION.md`,
> `back-end/IEEE829_TEST_SUMMARY_REPORT.md`, `back-end/IEEE829_TEST_INCIDENT_REPORT.md`,
> `back-end/ISTQB_QUALITY_REPORT.md`, `back-end/ISO25010_QUALITY_REPORT.md`,
> `back-end/ISO29110_PROCESS_REPORT.md`, `back-end/QUALITY.md`, `back-end/quality/quality-codes.md`.

## Types of tests

### Unit
- **Tool:** JUnit 5 + AssertJ (Java: 01-identity, 02-authorization, 04-scheduling, 05-attendance),
  Jest + ts-jest (TypeScript/Fastify: 03-academic, 07-configuration, 08-notification, 09-quality),
  pytest + pytest-asyncio (Python/FastAPI: 06-biometric).
- **Minimum coverage:** ≥ 70% globally (IEEE 829 Test Plan acceptance criterion);
  **> 80% on critical paths** as an ISTQB exit criterion (threshold set after the maturity assessment).
- **What they test:** use cases and domain logic isolated through ports (hexagonal architecture).
  Existing examples: `ResetPasswordUseCaseImplTest`, `RefreshSessionUseCaseImplTest`,
  `UserActivationServiceTest` (identity); email/alert rendering handlers in notification
  (`sender.test.ts`, `render.test.ts`, `routes.alerts.test.ts` under `08-ms-notification/src`).
- **What they do NOT test:** a real database, network, Kafka, or other external services (these are
  replaced with in-memory doubles, e.g. `InMemoryRecoveryCodeAdapterTest` and
  `tests/support/in_memory_repositories.py` in biometric).

### Integration
- **Tool:** Spring Boot Test with H2/in-memory databases (risk mitigation in the Test Plan: "DB
  unavailable → tests with H2/SQLite in memory"), Supertest against the real Fastify app (03-academic
  has `supertest` + `testcontainers` available as dependencies), and httpx/TestClient in FastAPI (06-biometric).
- **What they test:** the service and complete layers (controller → use case → adapter) without
  external services; CRUD endpoints per entity according to the TC-xx-xxx matrix in `IEEE829_TEST_PLAN.md`.
  Examples: `RoleControllerTest` (02), `AttendanceRecordControllerTest` (05), `academic.test.ts` (03).
- **When they run:** on every PR (along with unit tests and stack-specific static linting).

### Contract (Contract Testing)
- **Tool:** Custom router contract tests with pytest + `fastapi.testclient`
  (`back-end/06-ms-biometric/tests/unit/web/test_router_contract.py`) against the real app, with
  driven ports replaced by in-memory doubles and MongoDB explicitly disabled
  (`BIOMETRIC_MONGO_ENABLED=false`). For the other services, each service's OpenAPI contracts
  (`SERVICE.md` / `quality-spec.yaml`) serve as the reference contract; Pact/Spring Cloud Contract
  is a recommended future enhancement.
- **What they test:** whether the provider meets consumer expectations: request/response shapes,
  status codes (201/200/204), uniform error envelopes (`{error, code, message,
  correlationId, timestamp, path}` as specified in `quality-codes.md`), and degraded health-check behavior.
- **When they run:** on every producer and consumer PR; impact analysis is required for PRs
  that change contracts (a rule adopted in the ISTQB report, CTFL 2).

### E2E (End-to-End)
- **Tool:** API-level E2E with Newman/Postman + curl against the full stack in Docker Compose
  (Kong gateway :8000 → 8 microservices + PostgreSQL), with a large seed of real data using
  `tools/seed-api-test-data` (2 schools, 1,600 students, 80 teachers, ~1,210 sessions,
  ~81k attendance records) to test front↔back communication at scale from Web and Mobile.
- **Performance:** k6 / wrk (criteria: P95 ≤ 200 ms, 5xx error rate ≤ 1%).
- **Security:** OWASP ZAP (basic security tests) + authentication bypass verification
  (reported in `IEEE829_TEST_SUMMARY_REPORT.md`: "Authentication bypass: Not possible ✅").
- **What they test:** complete user flows with the entire system running (CRUD per service,
  authentication through Kong with opaque-session auth + the global rate limit of 6000/min, attendance reports).
- **When they run:** before each deploy to staging. The seed is idempotent, and its exit code serves
  as a gate: `0` if everything passes (or only documented known bugs marked WARN), `1` for any other failure.

### Exploratory & manual
- Exploratory, risk-guided testing and review of ISO 25010 criteria (usability,
  functional suitability) on Web/Mobile; validation of findings from the instrument
  (`GET /api/v1/quality/characteristics`, 24 Likert items) after each milestone.

## Minimum coverage per service

| Service | Stack | Unit | Integration | Contract | Status in repo |
|---------|-------|------|-------------|----------|----------------|
| 01-ms-identity | Java Spring Boot | 80% (critical paths >80%) | Required | Recommended | 22 test files (use cases, domain, DTO, adapters) |
| 02-ms-authorization | Java Spring Boot | 80% | Required | Recommended | 19 test files (incl. `RoleControllerTest`) |
| 03-ms-academic | TS Fastify | 80% | Required (jest+supertest) | Recommended | 2 test files (`academic.test.ts` + supertest suite) |
| 04-ms-scheduling | Java Spring Boot | 70% | Required | Recommended | 10 test files (overlap-guard etc.) |
| 05-ms-attendance | Java Spring Boot | 85% (critical) | Required | Required | 15 test files (incl. `AttendanceRecordControllerTest`) |
| 06-ms-biometric | Python FastAPI | 80% | Required (pytest+httpx) | **Required** (router contract tests) | 7 test files (`tests/unit/web` + `tests/`) |
| 07-ms-configuration | TS Fastify | 70% | Required | Recommended | 3 test files (incl. `crud.repository.test.ts`) |
| 08-ms-notification | TS Fastify | 80% | Required (jest) | Recommended | 16 test files (routes, authGuard, actorScope, permissions, kafka consumer, postgres repos, email/webhook, entities) |
| 09-ms-quality | TS Fastify | 80% | Required | Recommended | 8 test files (ISO 25010/ISTQB/29110 instruments) |
| 99-api-gateway (Kong) | Kong OSS | — | Required (smoke: routes + session auth + rate limit) | — | kong/kong.yml verified in E2E |

> **Source of truth for pass/fail:** `.github/workflows/ci.yml` runs every test suite on each
> push/PR to `main`; the figures below are the last recorded cycle and may lag behind CI.

**Global target (IEEE 829 §1.4):** test pass rate ≥ 95%, coverage ≥ 70%, P95 ≤ 200 ms,
5xx errors ≤ 1%, successful CRUD tests 100%. Most recent run: **81 tests, 78 passed,
3 failed → 96.3%** (INC-001…003 incidents open in `IEEE829_TEST_INCIDENT_REPORT.md`).

## What runs in CI/CD vs. manual

| Stage | Runs automatically | Evidence/tool |
|-------|--------------------|---------------|
| On every PR | Stack-specific lint (ESLint, ruff/mypy, go vet, Checkstyle) + unit + integration + contract tests | jest, surefire, pytest, `go test` |
| On merge to develop | IEEE 829 CRUD suite per service + incident report | Newman/curl against Compose |
| Before staging deploy | E2E critical flows + large-volume seed + k6 (P95/5xx) + basic ZAP | seed exit code as gate |
| Continuous / on demand | ISO 25010 (product), ISTQB (testing maturity), ISO 29110 (process) assessments via `/api/v1/quality/*` | 09-ms-quality |
| Manual | Exploratory testing, Web/Mobile usability, incident closure with formal retesting | QA lead |

**Merge exit criteria (ISTQB §2.3):** ISTQB level ≥ Acceptable (≥3.0) and technical + management
chapters ≥ 3.5; coverage >80% on critical paths; requirement ↔ test case ↔ defect traceability
through `projectId`; every fixed defect requires retesting before closure.

## How we know a change did not break anything

1. Automated regression: the IEEE 829 suite (CRUD matrix per entity, TC-01…TC-08) runs in full
   on every merge; `limit/offset` pagination and soft delete mitigate volume-related regressions.
2. Impact analysis is required for PRs that modify contracts (API/schema).
3. Enhanced health checks (version, uptime, timestamp) + `/health/quality` with metrics
   consolidated per service.
4. ISO 9001 error codes normalized with `correlationId` for incident traceability.
5. IEEE 829 summary report updated after each cycle (run/passed/failed tests and pass rate).