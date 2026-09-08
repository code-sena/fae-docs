# ADR-002 — Modular Monolith with Schema-per-Context on a Single PostgreSQL Instance

- **ID:** ADR-002
- **Date:** 2026-09-03
- **Status:** Proposed
- **Authors:** [Diego Andrés Gutiérrez Nuñez, Juan David Arboleda Perdomo, Jonattan Steven Rizo Solano]

---

## Context

The system's relational data model already defines eight explicit bounded contexts — Identity,
Authorization, Academic, Attendance, Biometric, IoT, Audit, and Configuration — each grouped
with `TableGroup` and governed by an explicit rule stated in the model's documentation header:

> "FK solamente dentro del mismo bounded context. Entre contextos: IDs públicos sin FK.
> La consistencia cross-context se resuelve mediante API/eventos."

Two project-level decisions must be reconciled against this model:

1. The system's data strategy is **one monolithic database, using schemas** to separate
   contexts — not one database instance per service.
2. The system's services are defined as **the bounded contexts already present in the
   relational model** — i.e., eight logical units of responsibility.

Taken independently, these two statements point toward different architectural styles.
"One database per service" (the generic principle documented in `05-architecture/overview.md`
§5 P2, and in `06-data/models.md` §1) is a microservices-era default that assumes each
service is an independently deployable process with its own database instance. A single
shared database instance contradicts that default directly — `05-architecture/pattern-guide.md`
even lists "Shared database" as a named anti-pattern for microservices ("Destroys service
independence").

The team is three SENA apprentices building an attendance system for a single educational
institution context. Operating eight independently deployed services — eight CI/CD
pipelines, eight sets of environment variables, eight health checks, eight runbooks — is a
disproportionate operational cost relative to the project's scope, team size, and expected
load (one institution, not a multi-tenant platform at scale).

**Known constraints:**
- Team of 3 developers with no dedicated DevOps role.
- Single educational institution as the initial deployment target (see `School` as an
  identity root entity in the model).
- No current requirement for independent scaling or independent deployment cadence per
  bounded context.

---

## Decision

**We decided:** The system is built as a **Modular Monolith**. Each bounded context already
identified in the relational data model (Identity, Authorization, Academic, Attendance,
Biometric, IoT, Audit, Configuration) is implemented as an internal **module** — a folder with its own
`domain/`, `application/`, and `infrastructure/` layers per the project's Hexagonal
Architecture standard (`05-architecture/hexagonal-architecture.md`) — rather than as a
separately deployed process.

All modules run inside a **single deployable application** connected to a **single
PostgreSQL instance**. Each bounded context owns **its own schema** inside that instance
(`identity`, `authorization`, `academic`, `attendance`, `iot`, `audit`, `configuration`;
`biometric` is addressed separately in ADR-003). Each schema has its own database role with
privileges restricted to that schema only — enforcing isolation at the database-permission
level, not only by team convention.

**Derived rule — cross-schema isolation:** No query may join tables or declare foreign keys
across schemas belonging to different bounded contexts. Cross-context references use public
IDs only (exactly as already stated in the data model's documentation header) and are resolved through
in-process application calls between modules — never through SQL joins. This rule exists so
that a schema can later be extracted into its own physical database (via the Strangler Fig
pattern already documented in `pattern-guide.md`) without discovering hidden SQL-level
coupling.

**Derived rule — module boundaries in code:** The Hexagonal Architecture dependency rule
(`infrastructure → application → domain`) applies *within* each module, plus one addition
specific to this project: **the `domain/` layer of one module may never import from the
`domain/` layer of another module.** Cross-module interaction happens only through each
module's input ports (application layer), mirroring the same discipline that would be
required if modules were separate services communicating over the network.

**Justification:** A single deployable process with schema-level isolation gives the team
most of the design benefits of bounded contexts (clear ownership, low coupling, ability to
reason about one context at a time) without the operational overhead of distributed systems
that the team and project scope do not justify today. The schema boundaries and the
no-cross-schema-join rule preserve the option to extract any context into an independent
service later, at the point where independent scaling or independent deployment actually
becomes necessary.

---

## Evaluated alternatives

| Alternative | Pros | Cons | Reason for discarding |
|------------|------|------|-----------------------|
| **Modular Monolith, one instance, schema-per-context (chosen)** | Low operational overhead; one deploy pipeline; transactions can span contexts when genuinely needed; preserves future extraction path via clear schema boundaries | Single point of failure and resource contention at the database level; team must enforce module boundaries by discipline plus DB roles, since there is no network boundary forcing it | — (chosen) |
| 8 independent microservices, database per service | Full deployment and scaling independence per context; matches the "textbook" microservices pattern already documented in `pattern-guide.md` | 8 CI/CD pipelines, 8 sets of infra, 8 health checks/runbooks to operate; requires network communication (REST/events) even for trivial cross-context reads; team of 3 has no dedicated DevOps capacity to sustain this | Operational cost is disproportionate to team size and project scope; no current requirement for independent scaling justifies the overhead |
| Single monolith, no internal schema/module boundaries | Simplest possible setup; fastest initial development | Directly contradicts the bounded contexts already established in the relational data model; domain terms and invariants from different contexts would mix (e.g., `Attendance` logic touching `Academic` tables directly); makes future extraction effectively impossible without a full rewrite | Would discard the domain modeling work already done and documented in `02-domain/domain-map.md` and the entity-relationship model |

---

## Consequences

**Positive:**
- One deployment pipeline, one set of environment variables, one health check to monitor —
  matches the team's actual operational capacity.
- Transactions that span multiple bounded contexts (see ADR-004) can use standard ACID
  guarantees where appropriate, since all schemas live in the same instance.
- Bounded context boundaries are preserved as schema and module boundaries, keeping the
  domain model in `02-domain/domain-map.md` and the entity-relationship model meaningful in the codebase.
- The path to extracting a high-load or independently-evolving context (e.g., Attendance)
  into its own service later remains open, via the Strangler Fig pattern.

**Negative / Trade-offs:**
- The single PostgreSQL instance is a single point of failure for the entire system; a
  crash or heavy query in one schema can degrade performance for all others.
- Isolation between modules depends on discipline (code review, DB role restrictions) rather
  than a hard network boundary — a careless developer *can* technically write a cross-schema
  join unless caught in review.
- Independent scaling of a single hot context (e.g., high-volume facial scans during peak
  attendance hours) is not possible without extracting that context first.

**Impact on the system:**
- Affected services: all eight bounded contexts.
- Documents that must be updated:
  - `05-architecture/overview.md` §5 (P2) and §4 (Service Catalog) — must be revised to
    state "schema-per-context in a single instance, see ADR-002" instead of the generic
    "Database per Service" principle.
  - `06-data/models.md` §"Data modeling principles" #1 — must reference ADR-002 instead of
    asserting one database per service unconditionally.
  - `09-microservices/service-catalog.md` — the "Local port" / "DB" columns must be
    reinterpreted as "N/A — internal module, see ADR-002" until/unless a context is
    extracted.
  - `05-architecture/pattern-guide.md` — the "Patterns adopted in this project" table should
    mark "Database per Service" as "No — see ADR-002 (schema-per-context variant)".

---

## Risks

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|-----------|
| A developer writes a cross-schema JOIN or FK "because it's technically possible in the same instance" | Medium | High (silently destroys future extractability) | Enforce via DB role permissions (a module's role cannot even see other schemas' tables) plus code review checklist item; see cross-reference in ADR-004 |
| Single instance becomes a resource contention bottleneck as usage grows | Low (given current single-institution scope) | Medium | Monitor per-schema resource usage; connection pool limits per DB role; documented as a trigger condition for revisiting this ADR |
| Module boundary in code erodes over time (imports across `domain/` folders) | Medium | Medium | Lint rule / import boundary check in CI, if tooling allows; code review checklist item |

---

## References

- Domain model and bounded context rules → the system's entity-relationship model (documentation header)
- Bounded contexts → `02-domain/domain-map.md`
- Hexagonal Architecture standard → `05-architecture/hexagonal-architecture.md`
- Strangler Fig pattern (future extraction path) → `05-architecture/pattern-guide.md`
- Related to: ADR-003 (Biometric persistence), ADR-004 (cross-context consistency)