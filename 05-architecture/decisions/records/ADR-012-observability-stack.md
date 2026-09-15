# ADR-012 — Observability Stack

- **ID:** ADR-012
- **Date:** 2026-09-12
- **Status:** Proposed
- **Authors:** FaceAttend EDU Team

---

## Context

NFR-007 requires structured logging, RED metrics, distributed tracing, and alerting from day one. The observability document (`13-operations/observability.md`) already describes the desired pillars (logs, metrics, traces) and tool suggestions (Prometheus, Grafana, Jaeger, OpenTelemetry) but treats them as recommendations, not formal decisions.

The observability stack must:
1. Collect structured JSON logs from all modules (TypeScript and Python).
2. Expose RED metrics (Rate, Errors, Duration) per endpoint.
3. Provide distributed tracing across the attendance registration flow (Biometric → Attendance → Audit).
4. Support alerting when SLIs violate SLOs.
5. Be operationally manageable by a 3-person team.

**Known constraints:**
- Modular monolith (ADR-002): all modules run in one process (except biometric-service which is Python).
- Biometric-service (Python) must integrate with the same observability stack.
- Local development requires running the full observability stack via Docker Compose.

---

## Decision

**We decided:** Use the **OpenTelemetry + Prometheus + Grafana + Jaeger** stack, with **pino** (TypeScript) and **structlog** (Python) for structured logging.

### Component selection

| Pillar | Tool | Purpose |
|--------|------|---------|
| **Structured Logging** | **pino** (TS) / **structlog** (Python) | High-performance structured JSON logging |
| **Metrics** | **Prometheus** + **prom-client** (TS) / **prometheus_client** (Python) | RED metrics collection and storage |
| **Tracing** | **OpenTelemetry SDK** + **Jaeger** | Distributed tracing across modules |
| **Dashboards** | **Grafana** | Visualization of metrics and traces |
| **Alerting** | **Alertmanager** (bundled with Prometheus) | Alert routing to Slack/email |

### Justification

1. **OpenTelemetry as the standard:** OpenTelemetry is the CNCF standard for observability instrumentation. Using it as the tracing and metrics foundation ensures vendor-neutral data collection.
2. **Prometheus for metrics:** Prometheus is the industry standard for cloud-native metrics. `prom-client` (TypeScript) and `prometheus_client` (Python) are mature clients.
3. **Jaeger for tracing:** Jaeger is the CNCF-distributed tracing backend. OpenTelemetry SDK exports traces to Jaeger natively.
4. **Grafana for dashboards:** Grafana visualizes both Prometheus metrics and Jaeger traces in a single UI.
5. **pino for TypeScript logging:** pino is the fastest Node.js logger; produces structured JSON output with minimal overhead.
6. **structlog for Python logging:** structlog is the standard structured logger for Python; pairs with the standard `logging` module.

**Instrumentation approach:**
- Each module creates its own logger instance with module-specific context (service name, version).
- HTTP middleware generates `correlationId` (UUID) per request, propagated through logs and traces.
- OpenTelemetry auto-instrumentation captures HTTP, PostgreSQL, Redis, and Kafka spans.
- Manual spans are created for critical business logic (e.g., biometric matching, attendance write).

---

## Evaluated alternatives

| Alternative | Pros | Cons | Reason for discarding |
|------------|------|------|-----------------------|
| **OpenTelemetry + Prometheus + Grafana + Jaeger (chosen)** | CNCF standard; vendor-neutral; free; mature ecosystem; handles all 3 pillars | More components to deploy; learning curve for OpenTelemetry API | — (chosen) |
| ELK Stack (Elasticsearch + Logstash + Kibana) | Excellent for log search; powerful querying | Heavy resource usage (Elasticsearch); no built-in metrics or tracing; separate tools needed for metrics/traces | Too heavy for 3-person team; only covers logging |
| Datadog / New Relic (SaaS) | Zero infrastructure; auto-instrumentation; beautiful UI | Expensive (per-host pricing); vendor lock-in; data leaves self-hosted infra | Cost and data sovereignty concerns |
| Winston-only logging | Already common in Node.js | No metrics or tracing; no correlation across modules; manual log search | Only covers one pillar; no alerting capability |

---

## Consequences

**Positive:**
- Single observability stack covers all 3 pillars (logs, metrics, traces).
- OpenTelemetry is vendor-neutral — can export to any backend.
- pino/structlog produce structured JSON for machine-parseable logs.
- Grafana provides a unified UI for metrics, traces, and log search.
- Correlation ID propagation enables end-to-end request tracing.

**Negative / Trade-offs:**
- Docker Compose stack grows by 4 containers (Prometheus, Grafana, Jaeger, Alertmanager).
- Team must learn OpenTelemetry instrumentation API.
- Jaeger storage grows with trace volume (requires retention configuration).

**Impact on the system:**
- Affected modules: all (every module emits logs, metrics, and traces).
- Documents that must be updated:
  - `13-operations/observability.md` — confirm selected tools; add configuration details
  - `10-devops/local-setup.md` — add observability containers to Docker Compose
  - `.env.example` — add `OTEL_EXPORTER_OTLP_ENDPOINT` value (already present, confirm Jaeger URL)
  - `09-microservices/services/*/runbook.md` — reference Grafana dashboards per service

---

## Risks

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|-----------|
| Observability stack consumes too many resources on dev machine | Medium | Low | Use lightweight configurations; reduce trace sampling rate in dev (10%); increase in staging (100%) |
| Team neglects instrumentation under deadline pressure | Medium | High | Add observability checklist to PR template; require metrics endpoint in service template; pair programming for initial instrumentation |
| Jaeger storage grows unbounded | Low | Medium | Configure trace retention (7 days dev, 30 days staging, 90 days production); monitor storage usage |

---

## References

- NFR-007 Observability → `04-requirements/non-functional.md`
- Observability design → `13-operations/observability.md`
- Incident management → `13-operations/incident-management.md`
- Correlation ID pattern → `13-operations/observability.md` §Correlation ID
