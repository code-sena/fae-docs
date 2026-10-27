# ADR-024 — Face matching metric and decision threshold (Euclidean distance on dlib 128-D)

- **ID:** ADR-024
- **Date:** 2026-10-26
- **Status:** Accepted
- **Decision:** supersedes the *cosine 0.85* part of ADR-014. The library choice in ADR-014 (OpenCV + `face_recognition`/dlib, 128-D) stays.
- **Authors:** FaceAttend EDU Team
- **Implementation note (verified against `FULL/back-end/06-ms-biometric`, 2026-10-26):** `domain/similarity.py`, `infrastructure/config/settings.py`.

---

## Context

ADR-014 specified a cosine threshold of 0.85. dlib encodings are not centered, so cosine scores between *different* people routinely exceed 0.85 (documented in `settings.py`). Using cosine on faces produced false acceptances. The jury question "which metric, which threshold, how do you balance FAR/FRR" needs a single recorded answer.

**Known constraints:**
- Model is dlib ResNet (128-D); it was trained for Euclidean distance (same person < ~0.6).
- Schools must be able to tune strictness without redeploying (Settings slider, 50–100 %).
- 1:N identify scan runs in the service over a Mongo `find` (ADR-014/ADR-003).

---

## Decision

**We decided:** faces are compared with **Euclidean distance**; the match threshold is a maximum distance.

- Default `face_match_max_distance = 0.5` (stricter than dlib's 0.6 to prefer low FAR).
- Score exposed to clients is `1 - distance` clamped to [0, 1] (`face_similarity`), stored as `match_score DECIMAL(5,4)`.
- School-level confidence slider (50–100 %) maps linearly to distance 0.60 → 0.35 (`face_threshold_from_confidence`); out-of-range values are clamped, so a bad setting can never be looser than 0.60 nor stricter than 0.35.
- Cosine stays only for generic vectors (`similarity_threshold = 0.85`); fingerprints use a keypoint-match **count** (`fingerprint_match_threshold = 8`).
- 1:N scan is bounded by `identify_scan_limit = 10 000`; reaching it logs a warning.

**FAR / FRR trade-off:** lowering the max distance lowers FAR and raises FRR (real learners rejected → retry or manual/justified attendance); raising it does the opposite. 0.5 is the default; 0.60 is the hard ceiling.

---

## Evaluated alternatives

| Alternative | Pros | Cons | Reason for discarding |
|---|---|---|---|
| Euclidean, max distance 0.5 — chosen | Metric the model was trained for; tunable; simple | Needs per-school tuning | — |
| Cosine ≥ 0.85 (original ADR-014) | Scale-free | Non-centered dlib vectors → false matches | Measured FAR unacceptable |
| MongoDB `$vectorSearch` | Server-side ANN | Atlas-only / extra infra | Scan is small (≤ 10 k); not needed |
| Pixel-by-pixel comparison | None | Breaks with light/pose | Not a valid biometric method |

---

## Consequences

**Positive:** documented, defensible answer for FAR/FRR; threshold changes need no code change.
**Negative / debt:**
- No measured FAR/FRR curve exists in the repo. **Action:** run an evaluation (genuine vs impostor pairs, with/without glasses) and attach the table here before the final defence.
- Twins/look-alikes are not evaluated; enrolment with several samples is the mitigation.
- No illumination pre-processing (CLAHE/histogram equalisation) exists; see ADR-025 limits.
