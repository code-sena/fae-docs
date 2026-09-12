# Design Decisions — Biometric Service

> Technical decisions specific to this service (not global architectural ones — those go in `05-architecture/decisions/`).
> Document here things that are not obvious: why this library, why this pattern, why this structure.

---

## DEC-001 Biometric data is stored independently from the identity model

**Date:** 2026-09-07
**Status:** Active

**Context:**
Facial and fingerprint data are large, vector-based, and operationally different from relational identity data. They require specialized storage and lifecycle handling.

**Decision:**
The biometric context persists canonical embeddings and templates in a document/vector-capable store, while keeping only subject IDs and metadata as references to external identity data.

**Alternatives considered:**
- Storing embeddings in the identity relational schema: rejected because it would mix identity concerns with heavy vector processing and limit independent evolution.

**Consequences:**
- ✅ Independent optimization for biometric workloads and indexing
- ⚠️ Requires explicit subject resolution and event-based coordination with identity and attendance services

---

## DEC-002 Subject references are public IDs, not local ownership links

**Date:** 2026-09-07
**Status:** Active

**Context:**
The service must correlate biometric templates with people without becoming responsible for the identity data model itself.

**Decision:**
`subject_id` is stored as a public identifier from the identity context, and the service treats it as an external reference rather than as a local ownership relation.

**Alternatives considered:**
- Direct foreign keys to a person table or identity database: rejected because it creates structural dependence on the identity schema.

**Consequences:**
- ✅ Biometric service remains portable and decoupled
- ⚠️ Matching and review workflows need explicit subject identity validation

---
