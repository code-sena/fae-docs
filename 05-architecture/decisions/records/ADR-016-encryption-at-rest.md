# ADR-016 — Encryption at Rest Strategy

- **ID:** ADR-016
- **Date:** 2026-09-12
- **Status:** Proposed
- **Authors:** FaceAttend EDU Team

---

## Context

NFR-004 requires sensitive data to be encrypted at rest. The sensitive data includes:
- Personal identification data (Person entity).
- Contact information.
- Attendance records.
- Justification supporting documents (stored in MinIO per ADR-009).
- Facial/fingerprint embeddings (stored in MongoDB per ADR-003).
- Biometric templates.
- Audit logs related to user changes.

The system must decide how to implement encryption at rest for each storage layer.

**Known constraints:**
- PostgreSQL stores most relational data (8 schemas per ADR-002).
- MongoDB stores biometric embeddings (ADR-003).
- MinIO stores justification documents (ADR-009).
- NFR-004 requires "sensitive data must be encrypted at rest."
- NFR-004 also requires "raw facial images should not be retained unless strictly necessary."
- Team of 3 with no dedicated security specialist.

---

## Decision

**We decided:** Use **PostgreSQL pgcrypto** for encrypting sensitive columns in PostgreSQL, combined with **MinIO server-side encryption** for document storage.

### Encryption approach per storage layer

| Storage | Mechanism | Scope | Implementation |
|---------|-----------|-------|----------------|
| **PostgreSQL** | **pgcrypto extension** | Sensitive columns (PII, attendance records) | `pgp_sym_encrypt()` / `pgp_sym_decrypt()` for specific columns |
| **MongoDB** | **MongoDB Field-Level Encryption (FLE)** | Biometric embeddings | Client-side encryption via MongoDB driver |
| **MinIO** | **Server-side encryption (SSE-S3)** | All objects (justification documents) | MinIO configuration: `MINIO_KMS_AUTO_ENCRYPTION=on` |
| **Filesystem** | **Not required** | N/A | PostgreSQL and MongoDB handle their own encryption |

### PostgreSQL pgcrypto implementation

```sql
-- Enable pgcrypto extension
CREATE EXTENSION IF NOT EXISTS pgcrypto;

-- Example: encrypting a sensitive column
ALTER TABLE identity.person
  ADD COLUMN encrypted_document BYTEA;

-- Encrypt on insert
UPDATE identity.person
SET encrypted_document = pgp_sym_encrypt(
  '1234567890',           -- plaintext
  'encryption_key_from_vault'  -- key from environment/vault
);

-- Decrypt on read
SELECT pgp_sym_encrypt(
  encrypted_document,
  'encryption_key_from_vault'
) AS document_number
FROM identity.person;
```

**Columns to encrypt:**
| Table | Column | Rationale |
|-------|--------|-----------|
| `identity.person` | `document_number` | National ID number — PII |
| `identity.person` | `phone` | Contact information — PII |
| `identity.person` | `email` | Contact information — PII |
| `attendance.attendance` | (full record) | Attendance records are sensitive per NFR-004 |

**What NOT to encrypt with pgcrypto:**
| Data | Reason |
|------|--------|
| User passwords | Already hashed with bcrypt (ADR-008) |
| User names | Needed for display and search; low sensitivity compared to document numbers |
| UUIDs/IDs | Not sensitive; needed for joins and lookups |
| Audit logs | Must be searchable; integrity ensured by append-only pattern |

### Key management

- Encryption keys are stored in environment variables, injected by the orchestrator (per `security-policy.md` §Secret management).
- Keys are **not** stored in source code, `.env` files, or logs.
- In production: keys are managed by the deployment platform (Docker secrets, Kubernetes secrets, or Vault).
- Key rotation: every 6 months or immediately if compromise is suspected (per `security-policy.md`).

---

## Evaluated alternatives

| Alternative | Pros | Cons | Reason for discarding |
|------------|------|------|-----------------------|
| **PostgreSQL pgcrypto + MinIO SSE (chosen)** | Transparent column-level encryption; native PostgreSQL extension; MinIO handles document encryption; no application code changes for most queries | Performance overhead for encrypt/decrypt; encrypted columns cannot be indexed normally; key management is manual | — (chosen) |
| Application-level AES-256-GCM | Full control; independent of storage engine; portable | Must encrypt/decrypt in application code; more complex key management; risk of encrypting wrong data | Higher implementation complexity; pgcrypto is simpler for PostgreSQL |
| PostgreSQL TDE (Transparent Data Encryption) | Transparent to application; no code changes | Requires PostgreSQL Enterprise or third-party extension; encrypts entire database (not selective); performance overhead on all queries | Not available in PostgreSQL OSS; overkill for selective column encryption |
| Disk-level encryption (LUKS) | Transparent; no application changes | Encrypts entire disk; does not protect against SQL injection or unauthorized DB access; data visible to DBA | Does not satisfy NFR-004 column-level requirement |

---

## Consequences

**Positive:**
- Sensitive PII (document numbers, phone, email) is encrypted at the column level.
- Justification documents are encrypted at rest via MinIO SSE.
- Biometric embeddings are encrypted via MongoDB FLE.
- pgcrypto is a native PostgreSQL extension — no additional infrastructure.

**Negative / Trade-offs:**
- Encrypted columns cannot use standard B-tree indexes (range queries require decryption).
- Performance overhead for encrypt/decrypt operations (~5-10% for affected queries).
- Key management is a new operational concern (key rotation, secure storage).

**Impact on the system:**
- Affected modules: `identity-service` (PII encryption), `attendance-service` (record encryption), `biometric-service` (embedding encryption).
- Documents that must be updated:
  - `00-governance/security-policy.md` — reference pgcrypto for encryption at rest
  - `06-data/models.md` — mark encrypted columns
  - `10-devops/local-setup.md` — document pgcrypto extension setup
  - `.env.example` — add `APP_ENCRYPTION_KEY` variable (placeholder)
  - `09-microservices/services/05-biometric-service/data-model.md` — document MongoDB FLE configuration

---

## Risks

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|-----------|
| Encryption key lost or mismanaged | Low | High | Keys stored in vault/secrets manager; backup keys in secure location; document key rotation procedure |
| Performance degradation from pgcrypto | Low | Medium | Only encrypt truly sensitive columns; benchmark affected queries; consider caching decrypted values |
| Encrypted data cannot be searched | Medium | Medium | Accept for PII columns (document_number); use hashed indexes for exact-match lookups if needed |

---

## References

- NFR-004 Security → `04-requirements/non-functional.md`
- Security policy → `00-governance/security-policy.md` §Secret management
- ADR-009 (MinIO file storage) → `05-architecture/decisions/records/ADR-009-file-storage.md`
- ADR-003 (MongoDB biometric) → `05-architecture/decisions/records/ADR-003-biometric-persistence.md`
