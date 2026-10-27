# ADR-025 — Liveness detection by randomised challenge-response (anti-spoofing)

- **ID:** ADR-025
- **Date:** 2026-10-26
- **Status:** Accepted (with documented limits)
- **Authors:** FaceAttend EDU Team
- **Implementation note (verified 2026-10-26):** `06-ms-biometric/infrastructure/web/liveness.py`, `domain/face_encoding.py`, `settings.py`.

---

## Context

A printed photo or a phone screen shown to the camera must not mark attendance or enrol a face. No ADR recorded how this is prevented.

**Known constraints:** single RGB camera on a phone/kiosk; dlib landmarks available; team without a CV specialist; no depth/IR hardware.

---

## Decision

**We decided:** active liveness by **random gesture challenges**, verified server-side.

- Server issues a challenge of 1 (enrol) to 3 gestures chosen at random and shuffled from `blink, turn_left, turn_right, smile, nod, open_mouth` (never the same sequence twice in a row). Client announces which gestures it can render.
- Token is a hand-rolled **HMAC-SHA256** signed payload `{actions, step, exp}` (not a JWT), TTL `liveness_ttl_seconds = 120`, single-use per step, minimum `liveness_min_step_seconds = 1.0` between steps.
- Blink = Eye Aspect Ratio delta ≥ 0.06 (`BLINK_EAR_DELTA`) over a frame series; head turn/nod/smile from landmark geometry.
- **Frozen-frame check** (`_frames_are_frozen`) rejects looped still images.
- **Same-person check** across steps and the final image (`liveness_identity_threshold = 0.75`).
- Production must fail to start without `BIOMETRIC_LIVENESS_CHALLENGE_SECRET` (today it falls back to an ephemeral secret → see audit S-15).

---

## Evaluated alternatives

| Alternative | Pros | Cons | Reason |
|---|---|---|---|
| Random active challenges — chosen | Defeats static photo and fixed replay; no hardware | Can be beaten by a video/deepfake playing the right gestures | — |
| Passive texture/reflection CNN | No user action | Needs a trained model and dataset | Out of scope/time |
| Depth/IR camera | Strong | Hardware cost | Not available |
| None | Zero effort | Photo attack trivial | Unacceptable |

---

## Consequences

**Limits (must be stated in the defence, audit S-18):**
- Not certified against ISO 30107-3 PAD; a recorded video performing the requested gestures may pass.
- No illumination normalisation (CLAHE); very dark/backlit rooms increase failures. **Correction proposed:** add grayscale + CLAHE before `face_encoding`.
- `_consumed_tokens` is a per-process in-memory set: valid only with a single ms-biometric replica; use Redis if scaled.
- Challenge secret must be mandatory in prod.
