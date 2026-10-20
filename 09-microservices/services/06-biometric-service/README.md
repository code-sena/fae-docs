# Biometric Service

> Owns facial and fingerprint embeddings, plus the workflow for updating those templates and validating identities.

---

## Responsibility

> This bounded context stores biometric artifacts and their lifecycle status without depending on the identity schema directly.

---

## Bounded context ownership

| Entity | Description |
|--------|-------------|
| Facial_Embedding | Stores the current facial vector for a subject (MongoDB). |
| Fingerprint_Embedding | Stores the fingerprint template and finger metadata (MongoDB). |
| Biometric_Update_Case | Tracks a request to replace or revise an embedding (facial or fingerprint). |

> **Note:** `biometric_update_case` is a SQL table **logically owned by Biometric but physically hosted in the `configuration` schema** (documented debt — see AGENTS.md §4.6).

---

## Location in the architecture

| Field | Value |
|-------|-------|
| Number in catalog | 06 |
| Local port | 8086 |
| Repository | `back-end/06-ms-biometric` |
| DB engine | MongoDB (embeddings) + SQL case table (in `configuration` schema) |
| Communicates with | attendance-service |
| Consumed by | Attendance validation workflow |

---

## Local rules

- `person_id` is an external identity reference to the Identity context, not a local FK.
- Face and fingerprint data are separated into independent collections with their own indexes.
- `Biometric_Update_Case` tracks review state (`Pending/In_Review/Approved/Rejected`) externally to the identity and attendance services.
- The service may evolve independently from the relational identity model.

---

## Related documents

- [data-model.md](./data-model.md) — biometric schema with vector, subject, and review records
- [events.md](./events.md) — recognition and approval events between services
- [decisions.md](./decisions.md) — design decisions for vector storage and identity boundaries
- [runbook.md](./runbook.md) — operational procedures for model updates and incident handling
- [API Contract](../../../07-api/contracts/openapi/06-biometric-service.yaml)

