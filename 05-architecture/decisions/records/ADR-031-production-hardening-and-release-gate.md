# ADR-031 — Production hardening baseline and release gate (“definition of finished”)

- **ID:** ADR-031
- **Date:** 2026-10-26
- **Status:** Proposed
- **Authors:** FaceAttend EDU Team
- **Source:** `REVISION_AUDITORIA_2026-10-25.md`, `CHECKLIST_MEJORAS.md`, `REVISION_LICENCIAS_TERCEROS.md`.

---

## Context

The audit found secrets on disk, all ports bound to `0.0.0.0`, dev rate limits (`minute: 6000`), missing web security headers, cleartext mobile traffic, undefined data-protection documents for minors and biometrics, and an unresolved licence decision (Liquibase 5 FSL, MinIO AGPL, Redis/Mongo source-available, dlib model licence, `xlsx`).

---

## Decision

**We decided:** the project is considered *finished* when every 🔴 item and the 🟠 items below are closed or formally accepted as risk in this ADR.

| Gate | Checklist IDs |
|---|---|
| Secrets & exposure | S-01..S-05, S-07, S-08, S-14, S-15 |
| Client hardening | S-09..S-12 |
| Legal (Colombia) | L-01..L-06, L-10 |
| Licences | L-12 (pin Liquibase 4.x **or** accept FSL in writing), L-13 (Redis 7.2/Valkey, MinIO decision), L-14 |
| Operations | D-01, D-03, D-04 |
| Repo | C-07, C-12 |

Everything else (🟡/⚪) is backlog and does not block closure. Each accepted risk is appended here with owner and date.

---

## Alternatives

| Alternative | Reason for discarding |
|---|---|
| Declare done without gate | Defence questions on security/legal would expose gaps |
| Fix everything | Not feasible in the remaining time |

---

## Consequences

- One agreed checklist to present; items can be ticked in `CHECKLIST_MEJORAS.md`.
- This ADR moves to *Accepted* once the team signs the gate list.
