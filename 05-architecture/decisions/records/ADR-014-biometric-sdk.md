# ADR-014 — Biometric SDK/Library

- **ID:** ADR-014
- **Date:** 2026-09-12
- **Status:** Proposed
- **Implementation note (2026-10-17, verified against `FULL/`):** the decision is implemented (OpenCV + `face_recognition`/dlib, 128-D embeddings, MongoDB vector search), **but thresholds do not come from `configuration-service`**: they are environment variables — `similarity_threshold` (default 0.85 cosine, `06-ms-biometric/infrastructure/config/settings.py`) and `BIOMETRIC_FINGERPRINT_MATCH_THRESHOLD=8` (keypoint **count**, not cosine distance). The ADR's Euclidean < 0.6 table was replaced by a single cosine threshold in code.
- **Authors:** FaceAttend EDU Team

---

## Context

The biometric-service module (Python/FastAPI per ADR-005) must capture facial images, extract embeddings, and perform similarity search against stored embeddings to identify students during attendance registration. The SRS defines Use Case 2 ("Escaneo facial" [Facial Scan]) and RF 3.1.1 ("Facial recording") as core requirements.

The biometric pipeline must:
1. Detect faces in a captured image.
2. Extract a fixed-dimensional embedding vector from each face.
3. Search the MongoDB vector store (ADR-003) for the nearest matching embedding.
4. Return a match/no-match result with confidence score.
5. Respond within 5 seconds (NFR-001: facial verification < 5s).

**Known constraints:**
- Biometric-service uses Python + FastAPI (ADR-005) and MongoDB (ADR-003).
- The user chose OpenCV2 + NumPy as the general approach.
- NFR-004 requires minimizing raw facial image retention; embeddings preferred over raw images.
- Team of 3 with no ML/CV specialist.

---

## Decision

**We decided:** Use **OpenCV** for face detection, **face_recognition** (based on dlib) for face embedding extraction, and **NumPy** for vector operations, with MongoDB vector search for similarity matching.

### Pipeline components

| Step | Library | Purpose |
|------|---------|---------|
| **Face Detection** | OpenCV (Haar Cascade or DNN module) | Detect face regions in the captured image |
| **Face Alignment** | OpenCV + dlib (landmarks) | Normalize face orientation for consistent embedding |
| **Embedding Extraction** | `face_recognition` (dlib-based) | Extract 128-dimensional face embedding vector |
| **Vector Search** | MongoDB Vector Search ($vectorSearch) | Find nearest neighbor embedding in the collection |
| **Distance Calculation** | NumPy | Compute Euclidean distance or cosine similarity between embeddings |

**Justification:**

1. **`face_recognition` library:** The `face_recognition` library (by Adam Geitgey) is a high-level, well-documented Python library built on dlib. It provides:
   - `face_encodings()` — extracts 128-D face embeddings (OpenFace model).
   - `face_distance()` — computes Euclidean distance between embeddings.
   - Simple API: 3 lines of code to detect faces and extract embeddings.
   - 99.38% accuracy on the Labeled Faces in the Wild (LFW) benchmark.

2. **OpenCV for detection:** OpenCV's DNN face detector (Caffe model) is more robust than Haar cascades for varying lighting and angles. OpenCV handles image preprocessing (resize, color conversion, normalization).

3. **NumPy for vector math:** NumPy is already a dependency of OpenCV and face_recognition. Embedding comparison is a simple vector distance calculation.

4. **MongoDB vector search:** Per ADR-003, MongoDB with vector search indexing (HNSW) handles similarity queries natively. The 128-D embedding fits naturally in a MongoDB document.

**Face matching threshold:**
- Euclidean distance < 0.6: **Match** (same person).
- Distance 0.6–0.8: **Uncertain** (require manual review or re-scan).
- Distance > 0.8: **No match** (different person).
- Thresholds are configurable via `configuration-service` (RF 4.4).

---

## Evaluated alternatives

| Alternative | Pros | Cons | Reason for discarding |
|------------|------|------|-----------------------|
| **OpenCV + face_recognition + NumPy (chosen)** | Simple API; 128-D embeddings; good accuracy; well-documented; pure Python | dlib compilation can be complex; 128-D embedding is smaller than deep learning models | — (chosen) |
| DeepFace (wrapper around multiple models) | Multiple models (VGGFace, FaceNet, ArcFace); one API for all | Heavier dependency; slower inference; larger model files; more complex than needed | Overkill for single-institution attendance; higher resource usage |
| insightface (ArcFace) | State-of-the-art accuracy; 512-D embeddings; GPU acceleration | Requires GPU for acceptable performance; heavier model; more complex installation | Team has no GPU infrastructure; overkill for attendance use case |
| OpenCV DNN + custom model | Full control over model selection | Requires training or downloading models manually; no built-in embedding extraction; more code to maintain | face_recognition provides the same functionality with less code |
| AWS Rekognition (external API) | Highest accuracy; zero local infrastructure | Vendor lock-in; cost per request; network dependency; privacy concerns with biometric data leaving the institution | Data sovereignty; cost; network dependency |

---

## Consequences

**Positive:**
- Simple API (face_recognition) reduces development time significantly.
- 128-D embeddings are compact and fast to compare.
- OpenCV + face_recognition are pure Python (after dlib compilation) — consistent with the biometric-service stack.
- No GPU required — runs on CPU for single-institution scale.
- Raw images are processed and discarded; only embeddings are stored (NFR-004 compliance).

**Negative / Trade-offs:**
- dlib compilation requires C++ build tools during Docker image build.
- 128-D embedding is lower-dimensional than ArcFace (512-D) — slightly less discriminative for large populations (not an issue for a single institution).
- `face_recognition` library is maintained by a single author (Adam Geitgey) — not a corporate-backed project.

**Impact on the system:**
- Affected modules: `biometric-service` (face processing), `attendance-service` (consumes match results).
- Documents that must be updated:
  - `09-microservices/services/06-biometric-service/README.md` — document face_recognition as the embedding library
  - `09-microservices/services/06-biometric-service/data-model.md` — document 128-D embedding format in MongoDB
  - `10-devops/local-setup.md` — document dlib build dependencies for Docker
  - `02-domain/domain-map.md` §6 Biometric — reference the embedding format

---

## Risks

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|-----------|
| dlib compilation fails in Docker | Medium | Medium | Use multi-stage Docker build with build dependencies; document in `10-devops/local-setup.md`; pre-built wheel available for most platforms |
| Face recognition accuracy degrades with poor lighting/angles | Medium | Medium | Document camera requirements; NFR-005 requires "normal lighting and camera conditions"; provide alternative registration flow per RF 3.1.2 |
| face_recognition library becomes unmaintained | Low | Medium | The underlying dlib library is well-maintained; can migrate to insightface or deepface if needed; embedding format is portable |
| False positives/negatives in face matching | Medium | High | Configurable threshold via configuration-service; audit trail for all matches; manual review flow for uncertain matches |

---

## References

- ADR-003 (MongoDB for biometric) → `05-architecture/decisions/records/ADR-003-biometric-persistence.md`
- NFR-001 Performance (facial verification < 5s) → `04-requirements/non-functional.md`
- NFR-004 Security (biometric data protection) → `04-requirements/non-functional.md`
- RF 3.1.1 Facial recording → `02-domain/domain-map.md` §6 Biometric
- face_recognition library: https://github.com/ageitgey/face_recognition
