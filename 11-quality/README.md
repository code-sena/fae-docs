# 11 — Quality · Testing Strategy (FaceAttend EDU)

> Estrategia completa de pruebas del proyecto. Fuentes en el repo:
> `back-end/IEEE829_TEST_PLAN.md`, `back-end/IEEE829_TEST_CASE_SPECIFICATION.md`,
> `back-end/IEEE829_TEST_SUMMARY_REPORT.md`, `back-end/IEEE829_TEST_INCIDENT_REPORT.md`,
> `back-end/ISTQB_QUALITY_REPORT.md`, `back-end/ISO25010_QUALITY_REPORT.md`,
> `back-end/ISO29110_PROCESS_REPORT.md`, `back-end/QUALITY.md`, `back-end/quality/quality-codes.md`.

## Types of tests

### Unit
- **Tool:** JUnit 5 + AssertJ (Java: 01-identity, 02-authorization, 04-scheduling, 05-attendance),
  Jest + ts-jest (TypeScript/Fastify: 03-academic, 07-configuration, 09-quality),
  pytest + pytest-asyncio (Python/FastAPI: 06-biometric), Go testing (Go/Gin: 08-notification).
- **Minimum coverage:** ≥ 70% global (criterio de aceptación del Test Plan IEEE 829);
  **> 80% en rutas críticas** como criterio de salida ISTQB (umbral fijado tras la evaluación de madurez).
- **What they test:** use cases y lógica de dominio aislados mediante puertos (arquitectura hexagonal).
  Ejemplos reales: `ResetPasswordUseCaseImplTest`, `RefreshSessionUseCaseImplTest`,
  `UserActivationServiceTest` (identity); handlers y render de email/alertas en notification
  (`email_handler_test.go`, `alert_handler_test.go`, `render_test.go`).
- **What they do NOT test:** DB real, red, Kafka u otros servicios externos (se sustituyen por
  doubles in-memory, p. ej. `InMemoryRecoveryCodeAdapterTest` y
  `tests/support/in_memory_repositories.py` en biometric).

### Integration
- **Tool:** Spring Boot Test con H2/en memoria (riesgo mitigado en el Test Plan: "DB no disponible →
  Tests con H2/SQLite en memoria"), Supertest sobre la app Fastify real (03-academic usa
  `supertest` + `testcontainers` como dependencia disponible), httpx/TestClient en FastAPI (06-biometric).
- **What they test:** servicio + capas completas (controller → use case → adaptador) sin servicios
  externos; endpoints CRUD por entidad según la matriz TC-xx-xxx de `IEEE829_TEST_PLAN.md`.
  Ejemplos: `RoleControllerTest` (02), `AttendanceRecordControllerTest` (05), `academic.test.ts` (03).
- **When they run:** on every PR (junto con unit tests y lint estático por stack).

### Contract (Contract Testing)
- **Tool:** Router contract tests propios con pytest + `fastapi.testclient`
  (`back-end/06-ms-biometric/tests/unit/web/test_router_contract.py`) contra la app real con los
  puertos driven reemplazados por dobles in-memory y MongoDB explícitamente deshabilitado
  (`BIOMETRIC_MONGO_ENABLED=false`). Para el resto de servicios, los contratos OpenAPI por servicio
  (`SERVICE.md` / `quality-spec.yaml`) actúan como contrato de referencia; Pact/Spring Cloud Contract
  queda como evolución recomendada.
- **What they test:** que el proveedor cumple lo que el consumidor espera: shapes de request/response,
  códigos de estado (201/200/204), envolventes de error uniformes (`{error, code, message,
  correlationId, timestamp, path}` según `quality-codes.md`) y comportamiento de health-checks degradados.
- **When they run:** on every PR of the producer and consumer; análisis de impacto obligatorio en PRs
  que toquen contratos (regla adoptada en el informe ISTQB, CTFL 2).

### E2E (End-to-End)
- **Tool:** API-level E2E con Newman/Postman + curl sobre el stack completo en Docker Compose
  (gateway Kong :8080 → 8 microservicios + PostgreSQL), y seed masivo de datos reales con
  `tools/seed-api-test-data` (2 colegios, 1600 estudiantes, 80 docentes, ~1210 sesiones,
  ~81k registros de asistencia) para probar comunicación front↔back a volumen desde Web y Mobile.
- **Performance:** k6 / wrk (criterios: P95 ≤ 200 ms, tasa de error 5xx ≤ 1%).
- **Security:** OWASP ZAP (pruebas básicas de seguridad) + verificación de bypass de autenticación
  (reportada en `IEEE829_TEST_SUMMARY_REPORT.md`: "Authentication bypass: No posible ✅").
- **What they test:** flujos completos de usuario con todo el sistema corriendo (CRUD por servicio,
  autenticación vía Kong con JWT + rate-limit 100/min, reportes de asistencia).
- **When they run:** before each deploy to staging. El seed es idempotente y su exit-code sirve de
  gate: `0` si todo pasa (o solo known bugs documentados como WARN), `1` en cualquier otro fallo.

### Exploratory & manual
- Pruebas exploratorias guiadas por riesgo y revisión de criterios ISO 25010 (usabilidad,
  adecuación funcional) sobre Web/Mobile; validación de hallazgos del instrumento
  (`GET /api/v1/quality/characteristics`, 24 ítems Likert) tras cada hito.

## Minimum coverage per service

| Service | Stack | Unit | Integration | Contract | Status in repo |
|---------|-------|------|-------------|----------|----------------|
| 01-ms-identity | Java Spring Boot | 80% (críticas >80%) | Required | Recommended | 6+ test classes (use cases, domain, DTO, adapters) |
| 02-ms-authorization | Java Spring Boot | 80% | Required | Recommended | RoleControllerTest |
| 03-ms-academic | TS Fastify | 80% | Required (jest+supertest) | Recommended | academic.test.ts |
| 04-ms-scheduling | Java Spring Boot | 70% | Required | Recommended | Pendiente (deuda QA registrada) |
| 05-ms-attendance | Java Spring Boot | 85% (crítico) | Required | Required | AttendanceRecordControllerTest |
| 06-ms-biometric | Python FastAPI | 80% | Required (pytest+httpx) | **Required** (router contract tests) | tests/unit/web + tests/ |
| 07-ms-configuration | TS Fastify | 70% | Required | Recommended | Sin tests aún (cobertura mínima exigida en PR) |
| 08-ms-notification | Go Gin | 80% | Required (go test) | Recommended | 3 paquetes de handler/render tests |
| 09-ms-quality | TS Fastify | 80% | Required | Recommended | Instrumentos ISO 25010/ISTQB/29110 como CRUD |
| 99-api-gateway (Kong) | Kong OSS | — | Required (smoke: rutas + JWT + rate-limit) | — | kong/kong.yml verificado en E2E |

**Meta global (IEEE 829 §1.4):** tasa de éxito de pruebas ≥ 95%, cobertura ≥ 70%, P95 ≤ 200 ms,
errores 5xx ≤ 1%, pruebas CRUD exitosas 100%. Resultado último ejecutado: **81 pruebas, 78 pasadas,
3 fallidas → 96.3%** (incidentos INC-001…003 abiertos en `IEEE829_TEST_INCIDENT_REPORT.md`).

## What runs in CI/CD vs. manual

| Stage | Runs automatically | Evidence/tool |
|-------|--------------------|---------------|
| On every PR | Lint por stack (ESLint, ruff/mypy, go vet, Checkstyle) + unit + integration + contract | jest, surefire, pytest, `go test` |
| On merge to develop | Suite IEEE 829 CRUD por servicio + reporte de incidentes | Newman/curl contra compose |
| Before staging deploy | E2E de flujos críticos + seed a volumen + k6 (P95/5xx) + ZAP básico | seed exit-code como gate |
| Continuous / on demand | Evaluaciones ISO 25010 (producto), ISTQB (madurez de pruebas), ISO 29110 (proceso) vía `/api/v1/quality/*` | 09-ms-quality |
| Manual | Pruebas exploratorias, usabilidad Web/Mobile, cierre de incidentes con retest formal | QA lead |

**Criterios de salida para merge (ISTQB §2.3):** nivel ISTQB ≥ Aceptable (≥3.0) y capítulos
técnicas + gestión ≥ 3.5; cobertura >80% en rutas críticas; trazabilidad requisito ↔ caso ↔ defecto
vía `projectId`; todo defecto corregido requiere retest antes de cerrar.

## How we know a change did not break anything

1. Regresión automatizada: suite IEEE 829 (matriz CRUD por entidad TC-01…TC-08) se ejecuta completa
   en cada fusión; paginación `limit/offset` y soft-delete mitigan regresión por volumen.
2. Análisis de impacto obligatorio en PRs que modifican contratos (API/schema).
3. Health checks enriquecidos (versión, uptime, timestamp) + `/health/quality` con métricas
   consolidadas por servicio.
4. Códigos de error ISO 9001 normalizados con `correlationId` para trazabilidad de incidentes.
5. Reporte de resumen IEEE 829 actualizado tras cada ciclo (ejecutadas/pasadas/fallidas/tasa).