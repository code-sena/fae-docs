# Privacy and Consent (Ley 1581 de 2012)

> How FaceAttend EDU handles personal and biometric data under Colombia's Ley 1581 de 2012
> (Habeas Data), and which article each technical decision answers. Where the law sets the
> result but not the means, the choice comes from `security-rules.md`.
> Code-level detail lives in the project repository: `docs/PROTECCION_DATOS_LEY_1581.md`.

---

## 1. Data the system processes

| Data | Where it lives | Nature under Ley 1581 |
|---|---|---|
| Name, last name, document type | `identity.person` | Personal data |
| Document number, email, phone | `identity.person` (encrypted) | Personal data |
| Blood type | `identity.person` (encrypted) | Sensitive: health (art. 5) |
| Face template (128-value vector) | MongoDB `facial_embeddings` (encrypted) | Sensitive: biometric (art. 5) |
| Fingerprint sample | MongoDB `fingerprint_embeddings` (encrypted) | Sensitive: biometric (art. 5) |
| Attendance records and justification documents | `attendance.*`, MinIO (SSE-S3) | Personal data |

Face photos are never stored: the template is computed and the image discarded.

## 2. Privacy notice (art. 12)

The notice is versioned (`2026-10` at the time of writing) and shown in web and mobile. It states:
the data controller and contact channel, the purpose (attendance control and academic
follow-up), the data collected, that answering about sensitive data (biometrics, health) is
**optional**, the holder's rights (know, update, rectify, revoke, request deletion) and how to
exercise them.

Only the current version is accepted. When the text changes the version goes up and every user
accepts it again on their next login.

## 3. Prior, express and informed consent (arts. 6-a, 9, 17-b)

Table `identity.data_processing_consent` keeps one row per acceptance:

| Column | Meaning |
|---|---|
| `person_id` | the data holder |
| `policy_version` | notice version accepted |
| `accepted_at` | date and time |
| `biometric_accepted` | separate, optional authorization for biometric data |
| `channel` | `self` (the holder) or `assisted` (an administrator recorded it) |
| `recorded_by` | who recorded it |
| `biometric_revoked_at` | revocation date, if any |

Rows are never deleted on revocation: they are the proof of authorization the law requires
(art. 17-b). `GET /api/v1/persons/{id}/consent` shows it to the holder and to their school's
administrator.

**When consent is asked:**

| Moment | Rule |
|---|---|
| Account creation (`POST /api/v1/users/provision`) | rejected without consent for the current version; the administrator confirms the holder accepted (`assisted`) and, separately, whether they authorized biometrics |
| Public school sign-up (`POST /api/v1/onboarding/schools`) | the registering administrator accepts the notice (`self`) |
| Login | without acceptance of the current version, the notice blocks the app until accepted |
| Before biometric capture | web and mobile check the biometric authorization and do not open the camera or reader without it; the biometric service checks again and answers **428 `BIOMETRIC_CONSENT_REQUIRED`** (REST and WebSocket), failing closed with 503 if identity is unreachable |

Biometric consent is optional (art. 6-a): without it a person still uses the system and their
attendance is recorded by other means.

## 4. Revocation and deletion (art. 8-e)

From the profile (web and mobile):

1. `DELETE /api/v1/biometric/persons/{id}` permanently deletes every face and fingerprint
   template of the person (history included) and removes the person from match logs.
2. `DELETE /api/v1/persons/{id}/consent/biometric` records the revocation.

Any later biometric enrollment asks for authorization again. Only the holder or a reviewer of
the same school can delete biometric data.

## 5. Security of the data (arts. 4-g, 17-d)

AES-256-GCM at rest (team rule `security-rules.md` A02; the law sets no algorithm). See
ADR-016 for the per-store design and its implementation note.

| Store | Encrypted | Mechanism |
|---|---|---|
| PostgreSQL `identity.person` | document number, email, phone, blood type | JPA converter in identity (`enc:v1:` prefix) + HMAC-SHA256 blind indexes (`email_hash`, `document_hash`) for login and uniqueness |
| MongoDB | face vector, fingerprint sample | AES-256-GCM in the biometric service, document `_id` as associated data |
| MinIO | every supporting document | SSE-S3 (`MINIO_KMS_SECRET_KEY`) |

A direct `psql` or `mongosh` query returns `enc:v1:…` values, not the data. Keys come from the
environment (`PII_ENCRYPTION_KEY`, `PII_HASH_KEY`, `BIOMETRIC_ENCRYPTION_KEY`,
`MINIO_KMS_SECRET_KEY`) and must be backed up apart from the data: without them the data cannot
be read.

## 6. Existing data

- Identity encrypts and indexes older rows at startup (`PiiBackfillRunner`, idempotent).
- Biometric templates: run `scripts/encrypt_existing_biometrics.py` once after deploying
  (`--dry-run` only counts). Old documents stay readable meanwhile.

## 7. Known limits

- Some staff biometric endpoints still return the decrypted vector (`template_response`);
  narrowing that is a separate API change.
- Database backups hold already-encrypted values, but the backup file itself is not encrypted.

## Correlations

- Security policy → `security-policy.md` §Data protection
- Encryption at rest → `05-architecture/decisions/records/ADR-016-encryption-at-rest.md`
- Document storage → `05-architecture/decisions/records/ADR-009-file-storage.md`
- NFR-004 Security → `04-requirements/non-functional.md`
