# Risk Register

> Unmitigated risks become problems. This register allows the team to
> anticipate, not just react.
> Review and update at each retrospective or when there is a significant project change.

---

## Probability × impact matrix

```
IMPACT
  │
  │ HIGH  │  [Mitigate]    │  [Avoid]       │
  │       │  Low           │  High          │
  │       │  Probability   │  Probability   │
  │───────────────────────────────────────
  │ MEDIUM│  [Accept]      │  [Mitigate]    │
  │       │  with monitoring│              │
  │───────────────────────────────────────
  │ LOW   │  [Accept]      │  [Accept]      │
  │       │                │               │
  └───────────────────────────────────────
              LOW              HIGH
                      PROBABILITY
```

**Response strategies:**
- **Avoid:** Change the plan so the risk cannot materialize
- **Mitigate:** Reduce the probability or impact
- **Transfer:** Pass the risk to another party (insurance, provider, contract)
- **Accept:** Acknowledge the risk and have a contingency plan

---

## Active risk register

### R-001 — [Risk name]

| Field | Value |
|-------|-------|
| **ID** | R-001 |
| **Category** | [Technical / Business / Team / External] |
| **Description** | [What could go wrong] |
| **Probability** | [High / Medium / Low] |
| **Impact** | [High / Medium / Low] |
| **Risk level** | [Critical / High / Medium / Low] |
| **Strategy** | [Mitigate / Avoid / Transfer / Accept] |
| **Mitigation plan** | [What we do to reduce probability or impact] |
| **Contingency plan** | [What we do if the risk materializes] |
| **Trigger** | [Warning signal that the risk is about to occur] |
| **Owner** | [Name of the person responsible for monitoring this risk] |
| **Review date** | [date] |
| **Status** | [Active / Mitigated / Occurred / Closed] |

---

### R-002 — [Risk name]

| Field | Value |
|-------|-------|
| **ID** | R-002 |
| **Category** | [Technical] |
| **Description** | [E.g.: Dependency X may not be available by the integration date] |
| **Probability** | [Medium] |
| **Impact** | [High] |
| **Risk level** | [High] |
| **Strategy** | [Mitigate] |
| **Mitigation plan** | [Start conversation with provider 6 weeks ahead; design mock/stub just in case] |
| **Contingency plan** | [Develop with mock, launch without the integration, add it in the next iteration] |
| **Trigger** | [No response from provider in 2 weeks] |
| **Owner** | [Name] |
| **Review date** | [date] |
| **Status** | [Active] |

---

### R-003 — Services without their own authentication (03, 06, 07, 08, 09) — CLOSED, see lesson below

| Field | Value |
|-------|-------|
| **ID** | R-003 |
| **Category** | Technical |
| **Description** | **Corrected 2026-10-11, re-verified against `FULL/`:** this risk described services with no auth check of their own; that is no longer true. `03-ms-academic`, `06-ms-biometric`, `07-ms-configuration`, `08-ms-notification` and `09-ms-quality` each run their own session+RBAC guard (`registerAuthGuard` TS, `require_permission` Python — same contract as the Java `AuthTokenFilter`), applied to every `/api/**` route except `/health`. A direct call to a service's port (compose publishes them on `127.0.0.1:8083-8089`) still has to pass that service's own check; it is not an open bypass. See `07-api/authentication.md` §1. |
| **Probability** | Low |
| **Impact** | Medium |
| **Risk level** | Low |
| **Strategy** | Accept (residual risk only: 4 duplicated implementations can drift — tracked as `AT-004`, not as an open-auth risk) |
| **Mitigation plan** | Keep every internal port inside the Docker network as defense in depth regardless (compose already binds them to `BIND_IP=127.0.0.1`); extract a shared validation module per stack to remove the duplication (`AT-004`). |
| **Contingency plan** | If a change to one guard is not mirrored in the other three, a route could end up under- or over-protected — catch this with a contract test per service asserting every `/api/**` route 401s without a bearer. |
| **Trigger** | A new route shipped in one service's guard logic without the equivalent permission rule in the other three. |
| **Owner** | Tech Lead |
| **Review date** | 2026-10-14 |
| **Status** | Closed |

---

### R-004 — Biometric update case has two unsynced implementations

| Field | Value |
|-------|-------|
| **ID** | R-004 |
| **Category** | Technical |
| **Description** | `ms-configuration` (`/api/v1/biometric-update-cases`, Postgres, the data model's canonical store) and `ms-biometric` (`/api/v1/biometric/update-request*`, MongoDB) both implement "request + review a biometric update" as fully independent features — no HTTP call or event links them. A case filed through one is invisible through the other, so a reviewer working from only one endpoint sees an incomplete queue, and the Mongo side accepts a `FINGERPRINT` case with no `fingerNumber` at all (the Postgres side rejects that with 400). See `08-uml/diagrams/source/seq-biometric-update-duplication.mmd`. |
| **Probability** | Medium |
| **Impact** | Medium |
| **Risk level** | Medium |
| **Strategy** | Mitigate |
| **Mitigation plan** | Pick one canonical implementation (the data model already names `ms-configuration`/Postgres as canonical) and make the other either call it (thin proxy, as `endpoint-matrix.md` already — incorrectly — claims it does today) or be removed; in the meantime, document both endpoints as separate queues for any reviewer-facing UI. |
| **Contingency plan** | If a biometric update is approved on one side only, re-run the request manually on the other side once the canonical store is decided. |
| **Trigger** | A support ticket where a submitted update request "disappeared" — check which of the two endpoints the client actually called. |
| **Owner** | Tech Lead |
| **Review date** | 2026-10-14 |
| **Status** | Active |

---

## Common technical risks in microservices

These risks apply to almost all microservices projects. Evaluate which ones apply:

| Risk | Typical probability | Standard mitigation |
|------|--------------------|--------------------|
| Failure cascade (one service down brings everything down) | Medium | Circuit Breaker, timeout, fallback |
| Data inconsistency between services | High | Saga, Outbox, idempotent events |
| Network latency in synchronous calls | High | gRPC, cache, async where possible |
| Lost messages in the broker | Medium | At-least-once + idempotent consumers |
| Schema evolution breaks consumers | High | Version events, compatible changes first |
| Technical debt accumulation | High | DoD with minimum coverage, regular reviews |
| Premature over-engineering | Medium | Start simple, YAGNI, measure before optimizing |
| Sensitive data exposure in logs | Medium | Logging policy, PII masking, SAST |
| Configuration drift between environments | High | Infrastructure as Code, environment variables |

---

## Closed risks / Lessons learned

| ID | Risk | Outcome | Lesson |
|----|------|---------|--------|
| R-003 | Services without their own authentication | Did not occur — risk description was stale | Several docs (`authentication.md`, `overview.md` AT-004, this file) kept describing a gap that was already closed in `FULL/` by the time they were last read. When a risk cites specific source files as evidence, re-check those files against the current code before trusting an old "verified" date — docs drift even after being marked verified. |
| R-00X | [name] | [Occurred / Did not occur] | [What we learned] |

---

## Template for adding a new risk

```markdown
### R-00X — [Name]

| Field | Value |
|-------|-------|
| **ID** | R-00X |
| **Category** | |
| **Description** | |
| **Probability** | |
| **Impact** | |
| **Risk level** | |
| **Strategy** | |
| **Mitigation plan** | |
| **Contingency plan** | |
| **Trigger** | |
| **Owner** | |
| **Review date** | |
| **Status** | Active |
```

---

## Correlations

- External dependencies → `15-project-control/dependencies.md`
- Related open questions → `15-project-control/open-questions.md`
- Architectural decisions that mitigated risks → `05-architecture/decisions/`
