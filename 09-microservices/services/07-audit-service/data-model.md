# Data Model — Audit Service

> This service is the authoritative owner of immutable operational traces and system error records.
> It records actions and failures without owning the business state of other bounded contexts.

---

## Database engine

**Engine:** PostgreSQL
**Justification:** Audit storage requires stable indexing, append-friendly writes, and long-term traceability.

---

## Schema

### Table: Audit_Event

| Field | Type | Nullable | Description | Constraints |
|-------|------|----------|-------------|-------------|
| audit_event_id | BIGINT | No | Event identifier | PK, auto-increment |
| school_id | UUID | Yes | Owning school if applicable | Optional |
| actor_id | UUID | Yes | User or system actor | External reference |
| action | VARCHAR | No | Action performed | Not null |
| aggregate_type | VARCHAR | No | Domain aggregate type | Not null |
| aggregate_id | VARCHAR | Yes | Aggregate instance identifier | Optional |
| description | TEXT | Yes | Human-readable event description | Optional |
| source_ip | VARCHAR | Yes | Originating client IP | Optional |
| application | VARCHAR | Yes | Sending service or application | Optional |
| occurred_at | DATETIME | No | Event time | Not null |

**Indexes:**
- `idx_audit_actor` on `(actor_id)`
- `idx_audit_aggregate` on `(aggregate_type, aggregate_id)`
- `idx_audit_date` on `(occurred_at)`
- `idx_audit_school` on `(school_id)`

### Table: Error_Log

| Field | Type | Nullable | Description | Constraints |
|-------|------|----------|-------------|-------------|
| error_id | BIGINT | No | Error record identifier | PK, auto-increment |
| school_id | UUID | Yes | Owning school if applicable | Optional |
| user_id | UUID | Yes | Related user if known | External reference |
| error_type | VARCHAR | No | Classification of error | Not null |
| description | TEXT | No | Error details | Not null |
| source_ip | VARCHAR | Yes | IP associated with the failure | Optional |
| application | VARCHAR | Yes | App or service origin | Optional |
| occurred_at | DATETIME | No | Error timestamp | Not null |

**Indexes:**
- `idx_error_user` on `(user_id)`
- `idx_error_date` on `(occurred_at)`
- `idx_error_school` on `(school_id)`

---

## Relationships

- None from this service to application aggregates.
- `actor_id`, `user_id`, and `school_id` are all external identifiers, intentionally without cross-context FK enforcement.

---

## Modeling decisions

- **Immutable traceability:** audit records are append-only and intended for forensic review.
- **Loose coupling:** the Audit service does not depend on the integrity of user or business tables being available.
- **Operational visibility:** logs can be correlated by time, actor, aggregate, and service.

---

## Schema migration

**Tool:** Flyway
**Script location:** src/migrations/
**Naming convention:** V[NNN]__[description].sql

**Rollback policy:**
- Never delete audit rows as a normal maintenance operation.
- Use archival or retention policies for long-term storage reduction.
