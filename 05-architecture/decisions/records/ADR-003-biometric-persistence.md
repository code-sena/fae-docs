# ADR-003 — Biometric Context Persistence: External NoSQL Document Store

- **ID:** ADR-003
- **Date:** 2026-09-03
- **Status:** Proposed
- **Implementation note (2026-10-05, verified against `FULL/`):** implemented as decided — embeddings live in MongoDB (`facial_embedding`, `fingerprint_embedding`, `faceattend_biometric`), and `configuration.biometric_update_case` is created by `database/07-ms-configuration-db` (physical hosting in the `configuration` schema is documented debt, per `AGENTS.md` §4.6). Job `06-ms-biometric-db` also creates a `biometric` schema.
- **Authors:** [Diego Andrés Gutiérrez Nuñez, Juan David Arboleda Perdomo, Jonattan Steven Rizo Solano]

---

## Context

ADR-002 establishes a single PostgreSQL instance with one schema per bounded context. The
`Biometric` context, however, contains an internal contradiction in the current data model
that this ADR must resolve before implementation begins.

The comment block above the `Biometric` group in the system's relational data model states:

> "NOTE: this context persists in NoSQL (e.g. MongoDB), not in a relational DB. The
> tables below are a logical/documentary representation of the collections... DBML does not
> natively model document schemas, so here NO real FKs are assumed between
> collections."
> (Translated from the original Spanish data-model note.)

Yet the same tables (`Facial_Embedding`, `Fingerprint_Embedding`) declare their core column
with the type `vector`:

```
embedding vector [not null]
template  vector [not null, note: 'Biometric fingerprint template']
```

`vector` is not a generic or document-database type — it is specifically the column type
introduced by **pgvector**, a PostgreSQL extension for storing and querying embeddings
(including similarity search via cosine distance, L2 distance, or inner product). The model
was, in effect, already designed with a relational/pgvector implementation in mind, even
though its accompanying note still describes a NoSQL target. This is the concrete
contradiction that must be settled, not left ambiguous, before the Biometric module is built.

This decision directly interacts with ADR-002: keeping Biometric on an external NoSQL engine
would make it the one context that breaks the "single monolithic database" premise — similar
in spirit to how Redis is already an accepted, documented exception for session/cache data in
`identity-service` (see `09-microservices/services/01-identity-service/`), but here
applied to a context that is core to the system's primary use case (facial recognition
attendance).

**Known constraints:**
- The instruction guiding this project is "one monolithic database, using schemas" — stated
  without carve-outs for specific contexts.
- Facial and fingerprint embeddings require **similarity search** (nearest-neighbor / distance
  queries), not just key-value storage or plain document storage.
- The team has no existing operational experience running a second database engine in
  production.
- The original data model documentation explicitly specifies NoSQL/MongoDB for this context,
  suggesting the intent was always to use a document database for biometric data.

---

## Decision

**We decided:** The `Biometric` bounded context uses **MongoDB 7 for embeddings plus one SQL workflow table**. The `facial_embeddings` and
`fingerprint_embeddings` collections reside in a dedicated MongoDB
database with vector search indexing enabled for similarity queries on embedding vectors. The `biometric_update_case` workflow table is logically owned by Biometric and physically hosted in the `configuration` schema (DBML v7).

**Justification:** Biometric data (facial embeddings, fingerprint templates) has fundamentally
different storage and query patterns than relational data. Document databases naturally
represent embedding records as self-contained documents, and MongoDB's vector search
capabilities provide native similarity search (cosine similarity, euclidean distance) with
dedicated vector indexing (HNSW) that is purpose-built for high-dimensional vector workloads.
This approach:

- Aligns with the original data model documentation that explicitly states "this context
  persists in NoSQL (e.g. MongoDB)"
- Provides better performance characteristics for vector similarity search at scale compared to
  pgvector, which requires manual tuning of index parameters (ivfflat/hnsw)
- Follows the established precedent in this project: Redis is already an accepted exception for
  session/cache data in `identity-service` (see `09-microservices/services/01-identity-service/`)
- Removes the contradiction between the data model's `vector` column types (which imply pgvector)
  and the documentation note (which states NoSQL) by choosing the NoSQL path and documenting
  MongoDB as the target

**Trade-off acknowledged:** This introduces a second database engine to operate, back up, and
monitor. However, the operational cost is justified by:
1. The specialized nature of vector search workloads
2. MongoDB's mature ecosystem for document storage with vector search
3. The team's willingness to learn MongoDB operations for this specific use case
4. The fact that Biometric is a bounded context with clear boundaries and limited cross-context
   data dependencies

The data model's comment block for `Biometric` remains consistent with this decision — it
already states "this context persists in NoSQL (e.g. MongoDB)". The `vector` column types
in the DBML model are conceptual representations; the actual MongoDB implementation will use
`array` fields for embedding vectors with a vector search index.

---

## Evaluated alternatives

| Alternative | Pros | Cons | Reason for discarding |
|------------|------|------|-----------------------|
| **External NoSQL document store (MongoDB) (chosen)** | Documents map naturally to "one embedding record"; native vector search with HNSW indexing; mature ecosystem for document storage; aligns with original data model documentation; purpose-built for high-dimensional vector workloads | Introduces a second engine to operate, back up, and secure; team must learn MongoDB operations; slight divergence from "single monolithic database" instruction | — (chosen) |
| pgvector in the same PostgreSQL instance | Single engine for the whole system, consistent with ADR-002; native similarity search operators; one backup/monitoring/security surface | Vector similarity search at scale requires manual tuning of index parameters (ivfflat/hnsw); less mature vector search capabilities compared to MongoDB; contradicts original data model documentation that specifies NoSQL; pgvector indexing is more complex to optimize | Operational simplicity of MongoDB's vector search was preferred over staying within PostgreSQL for this specialized workload |
| Dedicated vector database (e.g., Pinecone, Weaviate) | Best-in-class ANN performance and tooling for pure vector workloads | Third engine to operate in addition to PostgreSQL; massive overkill for this project's scale; no other part of the system needs it | Scale and team capacity do not justify a dedicated vector engine |

---

## Consequences

**Positive:**
- Aligns with the original data model documentation that explicitly specifies NoSQL/MongoDB for
  the Biometric context.
- Provides native vector search capabilities purpose-built for high-dimensional embedding data,
  with HNSW indexing for approximate nearest-neighbor queries.
- Self-contained document structure naturally represents embedding records without requiring
  joins or complex relational mappings.
- Clear separation of concerns: Biometric data lives in a specialized store optimized for its
  access patterns, while relational data remains in PostgreSQL.
- Follows the established precedent of using external stores (Redis for sessions/cache) when
  a specific workload benefits from specialized infrastructure.

**Negative / Trade-offs:**
- Introduces a second database engine to operate, back up, monitor, and secure — increasing
  operational surface area compared to a single PostgreSQL instance.
- The team must learn MongoDB operations, including vector search index configuration and
  query patterns.
- Cross-context queries between Biometric and relational contexts (e.g., joining embedding
  metadata with Person data) must be handled at the application layer rather than via SQL joins.
- Data consistency between MongoDB and PostgreSQL is eventual, not ACID — this is acceptable
  for the Biometric context because embedding data is read-heavy and write-once.

**Impact on the system:**
- Affected services: `Biometric` module only (`facial_embeddings`, `fingerprint_embeddings`,
  `biometric_update_case` SQL hosted in `configuration`).
- Documents that must be updated:
  - `09-microservices/services/06-biometric-service/data-model.md` — document the MongoDB
    database, collections, and vector search index configuration.
  - `09-microservices/service-catalog.md` — ensure biometric-service correctly reflects
    MongoDB as its data store.
  - The system's relational data model — the `Biometric` `TableGroup` comment block already
    states NoSQL/MongoDB, which is now consistent with this ADR.
  - `10-devops/local-setup.md` — add MongoDB as a required service for local development.

---

## Risks

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|-----------|
| MongoDB vector search performance degrades as embedding collection grows | Low at current scale (single institution) | Medium | Use HNSW indexing for approximate nearest-neighbor queries; document scale thresholds in `13-operations/observability.md` as metrics to watch |
| Team underestimates MongoDB operational learning curve | Medium | Low | Time-box a spike/proof-of-concept before committing the full Biometric module implementation; pair with existing MongoDB expertise if available |
| Data inconsistency between MongoDB and PostgreSQL due to eventual consistency | Low (Biometric data is write-once, read-heavy) | Medium | Implement idempotent writes and reconciliation checks for critical cross-context operations; document consistency guarantees in `09-microservices/communication-patterns.md` |
| Future multi-institution scale outgrows MongoDB's practical limits for vector search | Low (out of current scope) | Medium | Documented here as the trigger condition to revisit this ADR and consider extracting Biometric to a dedicated vector database, consistent with the extraction path preserved by ADR-002 |

---

## References

- Column types and context note → the system's entity-relationship model (`Biometric` group)
- Data strategy this ADR must remain consistent with → ADR-002
- MongoDB vector search documentation (external): https://www.mongodb.com/docs/manual/core/vector-search/
- Related to: ADR-002 (schema-per-context strategy), ADR-004 (consistency in attendance flow)