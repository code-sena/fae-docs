# ADR-006 — Message Broker Selection

- **ID:** ADR-006
- **Date:** 2026-09-12
- **Status:** Proposed
- **Implementation note (2026-10-05, verified against `FULL/`):** Kafka is **live**, not just "ready" — a KRaft broker runs in compose (`:9092`) and producers are wired in the Java (`spring-boot-starter-kafka` / `KafkaTemplate`) and TS (`kafkajs`) services. What is still missing: consumers with retry/DLQ topics (none exist) and a documented topic inventory in the deployment.
- **Update (2026-10-23):** the platform has **one consuming service, two consumers** — `08-ms-notification`
  reads `attendance.justification.events` (group `notification-service`, raises the
  dashboard alert to the apprentice when a justification is approved or rejected) and `scheduling.class-session.closed` (group
  `notification-service-class-session`, records class absences and e-mails the instructor).
  This is **not new** for the justification flow: the Go implementation already had
  it and it was ported unchanged to kafkajs in the TypeScript rewrite (ADR-021). It has
  in-place retry with backoff but **still no DLQ and no outbox**, and every other topic remains
  published-and-unread. AT-006 stands for everything except those two flows.
- **Authors:** FaceAttend EDU Team

---

## Context

The system requires asynchronous communication between bounded contexts for cross-context state propagation (per `09-microservices/communication-patterns.md`). The event catalog (`09-microservices/event-catalog.md`) defines ~30 business events across 8 services. The broker must support:

1. Publish-subscribe pattern for multi-consumer events.
2. Dead Letter Queue (DLQ) per service for failed event handling.
3. At-least-once delivery with idempotent consumers.
4. Schema evolution for event payloads (Confluent Schema Registry is part of this decision).
5. Operationally manageable by a 3-person team with no dedicated DevOps.

**Known constraints:**
- ADR-002 establishes a modular monolith — events are currently in-process but must be ready for extraction to external broker.
- The system handles ~30 registrations/minute at peak (NFR-001) — not high throughput.
- Team has no existing Kafka or RabbitMQ operational experience.

---

## Decision

**We decided:** Use **Apache Kafka** as the message broker, with **Confluent Schema Registry** for event schema management.

**Justification:**

1. **Event catalog alignment:** The event catalog already defines topic-per-context naming (`identity.user.events`, `attendance.attendance.events`), which maps directly to Kafka topics.
2. **Schema Registry integration:** Confluent Schema Registry provides formal schema evolution, backward/forward compatibility checks, and a registry that services can query at runtime — critical for a system with ~30 event types that will evolve.
3. **Durability:** Kafka persists events on disk by default, providing natural traceability and replay capability.
4. **Future extraction path:** When a module is extracted from the monolith (ADR-002), Kafka topics become the natural inter-service communication channel without changing event schemas.
5. **Industry standard:** Kafka is the de facto standard for event-driven architectures; team members' learning investment has long-term career value.

**Implementation approach for modular monolith:**
- In the monolith phase, modules publish events to Kafka topics via `kafkajs` (TypeScript) or `confluent-kafka-python` (Python).
- Consumers are in-process listeners that poll Kafka topics.
- When a module is extracted to a separate service, the Kafka connection remains unchanged — only the deployment topology changes.

---

## Evaluated alternatives

| Alternative | Pros | Cons | Reason for discarding |
|------------|------|------|-----------------------|
| **Apache Kafka + Confluent Schema Registry (chosen)** | Durable event log; schema evolution; industry standard; natural extraction path; pairs with Confluent Schema Registry | Heavier to operate than RabbitMQ; overkill for current throughput; requires ZooKeeper or KRaft | — (chosen) |
| RabbitMQ | Simpler to operate; lighter resource usage; native DLQ and dead-letter routing; better for task queue patterns | No built-in schema registry; no event replay; messages deleted after consumption; less natural for event-sourcing patterns | Lacks durability and replay; no native schema evolution; less aligned with the event-catalog design |
| Redis Streams | Already using Redis for sessions; one fewer engine; lightweight pub/sub | No built-in schema registry; limited consumer group management compared to Kafka; less mature for event-driven architectures | Too limited for 30+ event types with schema evolution requirements |
| In-process event bus only (no external broker) | Zero operational overhead; simplest possible setup | No persistence; lost on process restart; cannot support future extraction; no DLQ | ADR-004 already uses in-process events for decoupled side effects; a real broker is needed for cross-context events that must survive restarts |

---

## Consequences

**Positive:**
- Formal schema evolution via Confluent Schema Registry prevents breaking changes in event payloads.
- Durable event log enables services to replay events and rebuild state.
- Natural extraction path: when a module becomes a separate service, Kafka topics are already the communication channel.
- `kafkajs` (TypeScript) and `confluent-kafka-python` are mature clients for the chosen stacks.

**Negative / Trade-offs:**
- Kafka requires more resources (disk, memory) than RabbitMQ for the same throughput.
- Team must learn Kafka concepts (topics, partitions, consumer groups, offsets).
- Local development requires running a Kafka broker (Docker Compose handles this).
- Schema Registry is an additional service to operate.

**Impact on the system:**
- Affected modules: all eight bounded contexts (producers and consumers).
- Documents that must be updated:
  - `09-microservices/communication-patterns.md` — specify Kafka as the broker; remove "Kafka / RabbitMQ" ambiguity
  - `09-microservices/service-catalog.md` — update communication type column
  - `10-devops/local-setup.md` — add Kafka + Schema Registry to Docker Compose
  - `09-microservices/event-catalog.md` — add schema reference per event
  - `02-domain/domain-events.md` — reference Kafka as the transport mechanism

---

## Risks

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|-----------|
| Kafka operational complexity overwhelms the team | Medium | Medium | Start with single-node Kafka in Docker Compose; use Confluent Platform Docker images which include Schema Registry; document setup in `10-devops/local-setup.md` |
| Overkill for current throughput (30 registrations/minute) | Low | Low | Kafka's resource overhead is acceptable for a single-node setup; the investment pays off when modules are extracted |
| Schema Registry becomes a single point of failure | Low | Medium | Schema Registry is stateless and can run as a replicated container; fallback: cached schemas in consumers |

---

## References

- Event catalog → `09-microservices/event-catalog.md`
- Communication patterns → `09-microservices/communication-patterns.md`
- Related to: ADR-002 (Modular Monolith — events are in-process but Kafka-ready), ADR-004 (in-process events for attendance side effects)
