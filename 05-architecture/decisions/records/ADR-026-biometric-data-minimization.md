# ADR-026 — Biometric data minimisation: embeddings only, encrypted, no stored photos

- **ID:** ADR-026
- **Date:** 2026-10-26
- **Status:** Accepted
- **Authors:** FaceAttend EDU Team
- **Related:** ADR-003, ADR-016 (this ADR corrects its MongoDB FLE claim), `FULL/docs/PROTECCION_DATOS_LEY_1581.md`.

---

## Context

Facial and fingerprint data are sensitive data under Colombian Law 1581/2012 (art. 5 and 6). NFR-004 requires not retaining raw images.

---

## Decision

**We decided:**
1. The biometric service never persists the captured image. The frame is decoded in memory (`decode_image` with byte and pixel caps), converted to a 128-D encoding and discarded. (Verified: no image write path in `06-ms-biometric` outside tests.)
2. Embeddings and fingerprint samples are stored in MongoDB **encrypted at application level with AES-256-GCM** (`infrastructure/security/biometric_cipher.py`, key `BIOMETRIC_ENCRYPTION_KEY`), not with MongoDB Field-Level Encryption as ADR-016 stated. A migration script exists for legacy plaintext rows (`scripts/encrypt_existing_biometrics.py`).
3. A 128-D dlib encoding is a lossy feature vector; the original face cannot be reconstructed from it by simple inversion. It is still personal data and treated as such (it can identify a person if matched).
4. Match logs expire after `match_log_retention_days = 90`.
5. Re-enrolment goes through `biometric_update_case` (Pending → In_Review → Approved/Rejected) with `requested_by`, `reviewed_by`.

---

## Alternatives

| Alternative | Reason for discarding |
|---|---|
| Keep photos for audit | Violates minimisation, high breach impact |
| MongoDB FLE (ADR-016) | Requires KMS/enterprise tooling; app-level AES-GCM met the need |
| Plain embeddings | Sensitive data unprotected |

---

## Consequences

- **Positive:** a database leak does not expose faces; key loss makes data unreadable → keys must be backed up separately (audit S-03, runbook D-03).
- **Gaps to close (audit L-01..L-05, S-16):** guardian authorisation for minors (`TI`), published data policy, DPIA, stop returning decrypted vectors from staff endpoints, encrypted backups.
- "Non-reversible" must be worded as *not practically reconstructible*, not as a cryptographic hash.
