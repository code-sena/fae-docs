# System Observability — FaceAttend-Edu

> **Current state (verified 2026-10-17):** observability today is **structured JSON logs +
> per-service healthchecks** (`docker compose logs -f [service]`, Kong access log on stdout,
> `/health` + `/health/ready` + `/health/quality` per service, Portainer on the EC2 host).
> **Prometheus, Grafana, Jaeger, Alertmanager and the OpenTelemetry collector are NOT deployed**
> — the metrics/tracing/alerting described below is the **target** model (ADR-012, still
> Proposed). Do not assume `/metrics` endpoints or PagerDuty routing exist.

> An unobservable system cannot be operated. Observability in FaceAttend-Edu is designed to support a polyglot architecture (Java, TS, Python) ensuring that regardless of the runtime, the operational visibility remains consistent.

---

## Observability Stack: FaceAttend-Edu

Given the polyglot nature of the project, we implement a standardized observability layer across different runtimes:

| Runtime | Logging | Metrics | Tracing |
|----------|----------|----------|----------|
| **Java (Spring Boot)** | Logback / SLF4J | Micrometer / Prometheus | OpenTelemetry Java Agent |
| **TypeScript (Fastify)**| Pino / Winston | prom-client | OpenTelemetry Node SDK |
| **Python (FastAPI)** | structlog / Logging | prometheus_client | OpenTelemetry Python SDK |
| **Gateway (Kong)** | Kong Log Plugins | Kong Prometheus Plugin | OpenTelemetry (Native) |

---

## Logs

### Standard Format (Structured JSON)

To allow centralized log aggregation (e.g., ELK or Loki), all microservices must produce structured JSON logs. This is critical for the **API Gateway (Kong)** to correlate requests across the `faceattend-app` network.

**Minimum Required Fields:**
```json
{
  "timestamp": "2026-10-08T14:30:00.123Z",
  "level": "ERROR",
  "service": "ms-attendance",
  "environment": "production",
  "correlationId": "550e8400-e29b-41d4-a716-446655440000",
  "message": "Failed to record attendance for student",
  "context": {
    "studentId": "uuid-123",
    "sessionId": "uuid-456",
    "errorCode": "BIOMETRIC_MISMATCH",
    "durationMs": 120
  }
}
```

### Log Level Policy for FaceAttend-Edu

| Level | Usage in FaceAttend-Edu | Example |
|-------|-------------------------|---------|
| `DEBUG` | Internal state, SQL queries (Disabled in Prod) | "Querying biometric_embeddings for user X" |
| `INFO` | High-level business events | "Attendance recorded for Session S-101" |
| `WARN` | Recoverable issues or anomalies | "Retry attempt 2 for Notification Service" |
| `ERROR` | Failures requiring intervention | "Cannot connect to MongoDB (ms-biometric)" |
| `FATAL` | Service startup failure | "Port 8085 already in use" |

### Data Privacy (PII)
Due to the nature of biometric data, **it is strictly forbidden** to log:
- Full facial/fingerprint embeddings.
- Plain-text passwords or JWT tokens.
- Unmasked personal identification numbers.

---

## Metrics

### RED Metrics for Microservices
Each of the 9 components (8 MS + Gateway) must expose a `/metrics` endpoint in Prometheus format.

| Metric | Type | FaceAttend-Edu Application |
|--------|------|-----------------------------|
| `http_requests_total` | Counter | Total requests per service (e.g., `ms-identity` vs `ms-academic`) |
| `http_request_duration_seconds`| Histogram | Latency of biometric matching in `ms-biometric` |
| `http_requests_in_flight` | Gauge | Concurrent requests hitting `ms-attendance` during peak hours |

### Business-Specific Metrics (KPIs)

| Business Metric | Type | Description |
|-----------------|------|-------------|
| `attendance_records_total` | Counter | Total successful check-ins recorded |
| `biometric_match_failures_total`| Counter | Number of failed facial/fingerprint matches |
| `notification_delivery_latency` | Histogram | Time from alert trigger to notification sent (ms) |
| `active_sessions_count` | Gauge | Current number of active user sessions in `ms-identity` |

---

## Distributed Tracing

Since a single user action (e.g., marking attendance) involves **Kong $\rightarrow$ ms-attendance $\rightarrow$ ms-biometric $\rightarrow$ ms-notification**, distributed tracing is mandatory.

### Trace Propagation Flow
1. **Kong Gateway:** Generates the initial `trace-id` and injects it into headers.
2. **Application Layer:** Each service (Java/TS/Python) extracts the ID and creates a child span.
3. **Data Layer:** Automatic instrumentation for PostgreSQL (via Hibernate/Drizzle/FastAPI) and MongoDB.

---

## Monitoring & Alerting

### Grafana Dashboard Strategy
- **Executive View:** High-level availability of the 8 services and Gateway throughput.
- **Biometric Depth:** Specific monitoring for `ms-biometric` (MongoDB latency and embedding processing time).
- **Infrastructure View:** Docker container health, CPU/RAM usage for JVM services (which have higher footprints).

### Standard Alert Rules

| Alert | Condition | Severity | Action |
|-------|-----------|----------|--------|
| `CriticalServiceDown` | Health check fails for `ms-identity` or `ms-attendance` | **P0** | Immediate PagerDuty alert |
| `BiometricLatencyHigh`| P95 latency > 2s for `ms-biometric` | **P1** | Slack alert #ops-biometric |
| `DBConnectionSaturation`| Connection pool > 90% in any MS | **P2** | Slack alert #ops-infra |
| `NotificationQueueLag`| Kafka lag > 1000 messages | **P2** | Slack alert #ops-notifications |

---

## Correlations

- **SLA/SLO Definition** $\rightarrow$ `13-operations/README.md`
- **Service Definitions** $\rightarrow$ `back-end/SERVICES.md`
- **Healthcheck Definitions** $\rightarrow$ `docker-compose.yml`
- **Incident Response** $\rightarrow$ `13-operations/incident-management.md`
