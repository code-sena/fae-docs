# Service Readiness Checklist

> Operational checklist that must be completed before releasing any service to production. Each item must be verified and signed off.

---

## 1. Health and observability

| # | Item | Owner | Status |
|---|------|-------|--------|
| 1.1 | Health check endpoint implemented (`/health` or `/actuator/health`) | | ☐ |
| 1.2 | Readiness probe returns correct status when service is ready to accept traffic | | ☐ |
| 1.3 | Liveness probe detects hung processes and restarts the container | | ☐ |
| 1.4 | Structured logging enabled (JSON format with correlation ID) | | ☐ |
| 1.5 | Log level configurable at runtime without redeployment | | ☐ |
| 1.6 | Metrics endpoint exposed (Prometheus or equivalent) | | ☐ |
| 1.7 | Key business metrics defined and dashboarded (request rate, error rate, latency) | | ☐ |
| 1.8 | Distributed tracing enabled (OpenTelemetry or equivalent) | | ☐ |

---

## 2. Testing

| # | Item | Owner | Status |
|---|------|-------|--------|
| 2.1 | Unit tests passing (minimum 80% line coverage for business logic) | | ☐ |
| 2.2 | Integration tests passing (database, message broker, external APIs) | | ☐ |
| 2.3 | Contract tests passing (API contracts verified against OpenAPI spec) | | ☐ |
| 2.4 | End-to-end tests passing for critical user flows | | ☐ |
| 2.5 | Load tests completed and within acceptable latency thresholds | | ☐ |
| 2.6 | Chaos / failure tests executed (service degrades gracefully under failure) | | ☐ |

---

## 3. Documentation

| # | Item | Owner | Status |
|---|------|-------|--------|
| 3.1 | README.md complete with service description, responsibility, and port | | ☐ |
| 3.2 | data-model.md updated with current schema and relationships | | ☐ |
| 3.3 | events.md updated with all published and consumed events | | ☐ |
| 3.4 | OpenAPI contract published and validated | | ☐ |
| 3.5 | Runbook.md complete with startup, shutdown, and common procedures | | ☐ |
| 3.6 | decisions.md updated with relevant architectural decisions | | ☐ |
| 3.7 | Cross-context references documented in service catalog | | ☐ |

---

## 4. Security

| # | Item | Owner | Status |
|---|------|-------|--------|
| 4.1 | Authentication enforced on all non-health endpoints | | ☐ |
| 4.2 | Authorization checks in place (role-based or permission-based) | | ☐ |
| 4.3 | Input validation on all API parameters | | ☐ |
| 4.4 | SQL injection prevention (parameterized queries or ORM) | | ☐ |
| 4.5 | Secrets stored in vault or environment variables, never in code | | ☐ |
| 4.6 | TLS termination at API Gateway or service level | | ☐ |
| 4.7 | Rate limiting configured | | ☐ |
| 4.8 | Dependency vulnerability scan completed (no critical CVEs) | | ☐ |

---

## 5. Performance

| # | Item | Owner | Status |
|---|------|-------|--------|
| 5.1 | P95 latency under 200ms for synchronous API endpoints | | ☐ |
| 5.2 | P99 latency under 500ms for synchronous API endpoints | | ☐ |
| 5.3 | Database connection pool sized correctly | | ☐ |
| 5.4 | Event publish latency under 50ms | | ☐ |
| 5.5 | Memory usage stable under sustained load (no leaks) | | ☐ |
| 5.6 | CPU usage within acceptable limits under expected peak load | | ☐ |

---

## 6. Deployment

| # | Item | Owner | Status |
|---|------|-------|--------|
| 6.1 | Docker image builds successfully from clean state | | ☐ |
| 6.2 | Environment variables documented and externalized | | ☐ |
| 6.3 | Database migrations tested and reversible | | ☐ |
| 6.4 | Zero-downtime deployment strategy verified (rolling update or blue-green) | | ☐ |
| 6.5 | Rollback procedure tested | | ☐ |
| 6.6 | Service registered in API Gateway with correct routing | | ☐ |
| 6.7 | DLQ configured and alerts set up for event processing failures | | ☐ |

---

## 7. Integration

| # | Item | Owner | Status |
|---|------|-------|--------|
| 7.1 | Upstream service dependencies available and healthy | | ☐ |
| 7.2 | Downstream consumers identified and notified of API changes | | ☐ |
| 7.3 | Event topics created and permissions configured | | ☐ |
| 7.4 | Cross-context public ID references validated | | ☐ |
| 7.5 | Idempotent event handlers implemented and tested | | ☐ |

---

## Sign-off

| Role | Name | Date | Signature |
|------|------|------|-----------|
| Service owner | | | |
| Tech lead | | | |
| Security reviewer | | | |
| Operations | | | |

---

## Correlations

- Service catalog → `09-microservices/service-catalog.md`
- Communication patterns → `09-microservices/communication-patterns.md`
- Runbook template → `09-microservices/_template/service/runbook.md`
