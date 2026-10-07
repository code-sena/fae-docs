# Data Model — Biometric Service

> This service is the authoritative owner of biometric recognition data.
> It persists templates and review records independently from the identity service.

---

## Database engine

**Engine:** MongoDB / document storage plus vector search capabilities
**Justification:** Biometric templates are naturally represented as collections with large payloads and vector indexes.

---

## Schema

### Enum: Facial_Update_Case_Status

| Value |
|-------|
| Pending |
| In_Review |
| Approved |
| Rejected |

### Collection: Facial_Embedding

| Field | Type | Nullable | Description | Constraints |
|-------|------|----------|-------------|-------------|
| embedding_id | UUID | No | Embedding identifier | PK |
| school_id | UUID | No | Owning school | Not null |
| subject_id | UUID | No | Identity subject ID | External reference |
| embedding | VECTOR | No | Vector representation | Not null |
| model_version | VARCHAR | No | Model version | Not null |
| is_active | BOOLEAN | No | Active template flag | Default true |
| created_at | DATETIME | No | Creation date | Not null |
| updated_at | DATETIME | Yes | Last update | Nullable |

**Indexes:**
- `idx_embedding_subject_active` on `(subject_id, is_active)`
- `idx_embedding_school` on `(school_id)`

### Collection: Fingerprint_Embedding

| Field | Type | Nullable | Description | Constraints |
|-------|------|----------|-------------|-------------|
| embedding_id | UUID | No | Fingerprint template identifier | PK |
| school_id | UUID | No | Owning school | Not null |
| subject_id | UUID | No | Identity subject ID | External reference |
| template | VECTOR | No | Fingerprint template | Not null |
| finger | VARCHAR | Yes | Finger name or side | Optional |
| model_version | VARCHAR | No | Model version | Not null |
| is_active | BOOLEAN | No | Template active flag | Default true |
| created_at | DATETIME | No | Creation date | Not null |
| updated_at | DATETIME | Yes | Last update | Nullable |

**Indexes:**
- `idx_fingerprint_subject_finger_active` on `(subject_id, finger, is_active)`
- `idx_fingerprint_school` on `(school_id)`

### Collection: Facial_Update_Case

| Field | Type | Nullable | Description | Constraints |
|-------|------|----------|-------------|-------------|
| case_id | UUID | No | Case identifier | PK |
| school_id | UUID | No | Owning school | Not null |
| subject_id | UUID | No | Subject being updated | External reference |
| current_embedding_id | UUID | Yes | Current embedding reference | Optional |
| reason | TEXT | No | Reason for case creation | Not null |
| status | Facial_Update_Case_Status | No | Case state | Default Pending |
| requested_by | UUID | Yes | Requesting user | External reference |
| requested_at | DATETIME | No | Request time | Not null |
| reviewed_by | UUID | Yes | Reviewing user | External reference |
| reviewed_at | DATETIME | Yes | Review date | Nullable |
| resolution_notes | VARCHAR | Yes | Reviewer notes | Optional |
| created_at | DATETIME | No | Creation time | Not null |
| updated_at | DATETIME | Yes | Last update | Nullable |

**Indexes:**
- `idx_facial_case_subject` on `(subject_id)`
- `idx_facial_case_school` on `(school_id)`
- `idx_facial_case_status` on `(status)`

---

## Relationships

- `Facial_Update_Case.subject_id` → external `Identity.Person.person_id` or equivalent public ID
- `Facial_Update_Case.current_embedding_id` → `Facial_Embedding.embedding_id` or related template document reference

> Since this bounded context is persisted in document storage, no real FK constraints are assumed between collections; IDs are resolved by the application layer.

---

## Modeling decisions

- **Independent identity boundary:** the biometric context does not own `Person`; it tracks a subject by public ID only.
- **NoSQL flexibility:** embedding arrays are stored as document payloads that support vector search and model evolution.
- **Review workflow:** `Facial_Update_Case` provides traceability for template updates and approval decisions.

---

## Schema migration

**Tool:** MongoDB change streams / migration script runner (versioned migrations)
**Script location:** src/migrations/
**Naming convention:** V[NNN]__[description].js

**Rollback policy:**
- Prefer non-destructive updates to embedding data.
- Maintain versioned historical templates when replacing active vectors.

