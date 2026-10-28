# ADR-029 — Attendance capture is online-first; offline queue is out of scope

- **ID:** ADR-029
- **Date:** 2026-10-26
- **Status:** Proposed (needs team confirmation before the defence)
- **Authors:** FaceAttend EDU Team
- **Implementation note (verified 2026-10-26):** no SQLite, no `PENDING_SYNC`, no offline queue in `FULL/front-end/Mobile`.

---

## Context

The jury asks what happens when the classroom loses Internet. Facial identification runs server-side (dlib in ms-biometric, DB in Mongo); an offline phone cannot identify anyone. A locally stored attendance would have no verified identity.

---

## Decision

**We decided (proposed):** attendance by face requires connectivity to the platform (LAN is enough; Internet is not, because the stack is deployable on-premises with Docker Compose). When the backend is unreachable:
- the app shows a retryable error without closing the session (API client does not clear credentials on network errors);
- the instructor registers attendance **manually** (`capture_method = MANUAL`) once back online, or via `IMPORT`.

An offline queue (`expo-sqlite`, `PENDING_SYNC`, background sync with idempotency on `uq_attendance_session_actor`) is recorded as **future work**.

---

## Alternatives

| Alternative | Pros | Cons |
|---|---|---|
| Online-first + manual fallback — chosen | Simple; no unverified identities | Needs LAN/server |
| On-device matching + SQLite queue | Works offline | Embeddings on every phone: privacy/security risk (ADR-026); heavy |
| Queue photos and match later | Offline capture | Retains raw images; violates ADR-026 |

---

## Consequences

- Be explicit in the defence: no offline mode is implemented. Do **not** claim `PENDING_SYNC`.
- Verify that network errors do not log the user out (`sessionRefresh`/`apiClient` tests).
- If the team wants the queue, it needs a new ADR superseding this one.
