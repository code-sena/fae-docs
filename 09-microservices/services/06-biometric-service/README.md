# Biometric Service

> Owns facial and fingerprint embeddings, plus the workflow for updating those templates and validating identities.

---

## Responsibility

> This bounded context stores biometric artifacts and their lifecycle status without depending on the identity schema directly.

---

## Bounded context ownership

| Entity | Description |
|--------|-------------|
| Facial_Embedding | Stores the current facial vector for a subject. |
| Fingerprint_Embedding | Stores the fingerprint template and finger metadata. |
| Facial_Update_Case | Tracks a request to replace or revise a facial embedding. |

---

## Location in the architecture

| Field | Value |
|-------|-------|
| Number in catalog | 05 |
| Local port | 8086 |
| Repository | [Service repo URL] |
| DB engine | MongoDB / document storage with vector capability |
| Communicates with | attendance-service |
| Consumed by | Attendance validation workflow |

---

## Local rules

- `subject_id` is an external identity reference to the Identity context, not a local FK.
- Face and fingerprint data are separated into independent collections with their own indexes.
- `Facial_Update_Case` tracks review state externally to the identity and attendance services.
- The service may evolve independently from the relational identity model.

---

## Related documents

- [data-model.md](./data-model.md) — biometric schema with vector, subject, and review records
- [events.md](./events.md) — recognition and approval events between services
- [decisions.md](./decisions.md) — design decisions for vector storage and identity boundaries
- [runbook.md](./runbook.md) — operational procedures for model updates and incident handling
- [API Contract](../../../07-api/contracts/openapi/06-biometric-service.yaml)

