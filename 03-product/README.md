# 03 — Product Definition

> **What is this?**  
> The answer to “What are we going to build?”. It is not “how” — that belongs to architecture and implementation.  
> For **FaceAttend EDU**, this section defines the validated problem, the product vision, and the build plan for an attendance-management solution based on facial recognition, alternative recording methods, traceable histories, and justification workflows.

---

## Why this section exists

Without a clear product definition:

- The team may build features nobody asked for.
- Scope may grow out of control.
- There is no clear way to know whether the project was successful.
- Requirements from the SRS may be implemented without a shared product direction.

This section is the agreement between the team and stakeholders about **what will be built, why it will be built, and what will be excluded during the current project cycle**.

For FaceAttend EDU, this section prevents the team from prematurely building advanced analytics, grading modules, payment features, or physical-access control before validating the core attendance problem.

---

## Product intent

This section translates the Software Requirements Specification (SRS) of FaceAttend EDU into product decisions.

The product definition is based on:

- The institutional problem described in the SRS: inefficient attendance validation and lack of effective tracking.
- The need to reduce instructors’ administrative workload.
- The need to improve attendance reliability and prevent inconsistencies.
- The need to provide traceable reports, justifications, and role-based visibility.
- The need to protect sensitive personal and biometric data.

---

## What is here and how it was filled for FaceAttend EDU

| Document | Status | Content summary |
|---|---|---|
| `problem-framing.md` | ✅ Complete (v1.1) | Problem in one sentence, JTBD for 3 personas, evidence vs assumptions, current solutions, hypothesis and risks, metrics with baselines and 6-month targets, out-of-scope explicitly defined |
| `vision.md` | ✅ Complete (v1.1) | Vision statement (Geoffrey Moore), personas, mission, 5 strategic pillars with success metrics, 3-horizon roadmap with MVP scope |
| `product-backlog.md` | 🔲 Planned | Will be seeded from `04-requirements/HU` after MVP validation |

> **Evaluation fix (2026-09):** This section was flagged as template-only. Placeholders were removed: `problem-framing.md` now has validated evidence vs pending research clearly separated, and `vision.md` has quantified North Star + horizon roadmap. All decisions trace to SRS RF/RNF.

## Product decision log

- **Audit as domain removed (2026-09):** Audit is now infra concern, not product pillar — removed from 02-domain/06-data/09-microservices.
- **IoT out-of-scope for MVP:** Kept as external supporting context in `09-microservices/services/_future-iot-service` for future hardware, not in core domain map.
- **Biometric choice:** FACIAL primary, FINGERPRINT optional per `Biometric_Update_Case` workflow (see `02-domain`).

## How to use this section in a sprint

1. Start from `problem-framing.md` §6 metrics — pick one metric to move.
2. Check `vision.md` pillars to ensure the story aligns.
3. Write HU in `04-requirements/HU` referencing RF and the pillar.
4. Update this doc if a product decision changes (same PR as code).