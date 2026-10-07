# ADR-009 — File Storage for Justification Documents

- **ID:** ADR-009
- **Date:** 2026-09-12
- **Status:** Proposed
- **Authors:** FaceAttend EDU Team

---

## Context

The attendance module requires students to upload supporting documents for justification requests (RF 6.1: "Upload justification supporting documents"). NFR-003 mentions "object storage or a dedicated file service" as the proposed strategy. NFR-010 requires these files to be included in the backup strategy.

The system must store:
- Justification supporting documents (PDF, images, DOCX — up to 5 MB per file, per NFR-001).
- Potentially facial images during biometric enrollment (per ADR-015, though raw images should be minimized per NFR-004).

**Known constraints:**
- NFR-004 requires sensitive data (including justification supports) to be encrypted at rest.
- NFR-010 requires files to be included in backup strategy.
- Single institution deployment — not expecting millions of files.
- Self-hosted infrastructure (no cloud provider dependency).
- Team has no dedicated DevOps role.

---

## Decision

**We decided:** Use **MinIO** (S3-compatible, self-hosted) as the object storage for justification documents and any other file uploads.

**Justification:**

1. **S3-compatible API:** MinIO implements the S3 API, meaning any S3 client or library works without modification. If the system ever migrates to cloud storage (AWS S3, GCS), the client code changes only the endpoint URL and credentials.
2. **Self-hosted:** No external dependency on a cloud provider. Runs as a Docker container alongside the application stack.
3. **Encryption at rest:** MinIO supports server-side encryption (SSE-S3, SSE-KMS) natively, satisfying NFR-004.
4. **Backup-friendly:** MinIO stores files on the local filesystem. Standard filesystem backup tools (rsync, restic, volume snapshots) capture both MinIO data and metadata.
5. **Lightweight:** Single binary, minimal resource usage. Appropriate for a single-institution deployment.
6. **No vendor lock-in:** The S3 API is an open standard supported by all major cloud providers.

**Implementation:**
- MinIO runs as a Docker container with a persistent volume mount.
- The `attendance-service` module uploads files via the MinIO SDK (`minio` npm package for TypeScript, `minio` pip package for Python).
- Files are organized by bucket: `justification-documents/`, `biometric-enrollment/`.
- Each file is named with a UUID to avoid collisions: `{bucket}/{year}/{month}/{uuid}.{ext}`.
- Metadata (uploading user, justification ID, timestamp) is stored in PostgreSQL alongside the justification record.

---

## Evaluated alternatives

| Alternative | Pros | Cons | Reason for discarding |
|------------|------|------|-----------------------|
| **MinIO (S3-compatible, self-hosted) (chosen)** | S3 API standard; self-hosted; encryption at rest; Docker-ready; no vendor lock-in | Requires running a Docker container; local filesystem storage (not distributed) | — (chosen) |
| Local filesystem + volume mount | Simplest possible; no additional service | No S3 API; no built-in encryption; harder to migrate to cloud; permission management is manual; backup requires filesystem-level tools | Not production-ready; no encryption; migration path is unclear |
| AWS S3 | Fully managed; virtually unlimited storage; built-in encryption; lifecycle policies | Vendor lock-in; requires AWS account and IAM configuration; cost per GB; network dependency; egress fees | Team operates self-hosted; vendor lock-in is undesirable |
| PostgreSQL Large Objects | No additional service; already using PostgreSQL | PostgreSQL is not designed for file storage; poor performance for large files; complicates backups; increases DB size significantly | Wrong tool for the job; PostgreSQL documentation advises against this pattern |
| GridFS (MongoDB) | Already using MongoDB for biometric; no additional service | GridFS is designed for files >16MB; adds complexity to MongoDB; couples file storage to the biometric database | Justification files are <5MB; coupling to MongoDB complicates backups and separation of concerns |

---

## Consequences

**Positive:**
- S3-compatible API provides a migration path to cloud storage without code changes.
- Self-hosted MinIO keeps all data under the team's control.
- Built-in encryption satisfies NFR-004 for justification documents.
- Docker Compose integration is straightforward with official MinIO images.

**Negative / Trade-offs:**
- MinIO adds one more Docker container to the development and production stack.
- Single-node MinIO is not horizontally scalable (acceptable for single-institution scope).
- Filesystem-level backups must include the MinIO data volume.

**Impact on the system:**
- Affected modules: `attendance-service` (file upload/download), `configuration-service` (biometric enrollment files).
- Documents that must be updated:
  - `10-devops/local-setup.md` — add MinIO container to Docker Compose
  - `.env.example` — add MinIO endpoint, access key, and secret key variables
  - `09-microservices/services/05-attendance-service/data-model.md` — document file reference storage pattern
  - `06-data/models.md` — document `Supporting_Document` entity's file reference pattern
  - `10-devops/environments.md` — add MinIO to environment configuration matrix

---

## Risks

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|-----------|
| MinIO data volume runs out of disk space | Low (single institution, <5MB files) | Medium | Monitor disk usage via MinIO metrics; set up alerts at 80% capacity; document volume sizing in `13-operations/observability.md` |
| MinIO container crashes and files are lost | Low | High | Persistent volume mount ensures files survive container restarts; include MinIO volume in backup strategy (NFR-010) |
| S3 API incompatibility with future cloud migration | Very Low | Low | MinIO's S3 compatibility is well-tested; test with AWS S3 client before migration |

---

## References

- NFR-003 Scalability → `04-requirements/non-functional.md`
- NFR-004 Security (encryption at rest) → `04-requirements/non-functional.md`
- NFR-010 Disaster Recovery (backup) → `04-requirements/non-functional.md`
- RF 6.1 Justification upload → `02-domain/domain-map.md` §5 Attendance
- Supporting documents entity → `06-data/models.md`
