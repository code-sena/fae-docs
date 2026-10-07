# Storage and Documents

> File, document, and media storage boundaries per service. Defines what each service stores, where it is stored, and the applicable size and retention limits.

---

## 1. Storage strategy overview

| Storage type | Use case | Technology |
|-------------|----------|------------|
| Relational DB | Structured business data | PostgreSQL |
| Document DB | Semi-structured data with flexible schema | MongoDB |
| Cache | Session data, temporary tokens | Redis |
| Object storage | Files, documents, images, biometric templates | S3-compatible (MinIO / AWS S3) |
| Local filesystem | Temporary processing artifacts | Ephemeral container volume |

---

## 2. Storage per service

### Identity Service (+ auth merged)

| Asset type | Storage | Retention | Max size |
|-----------|---------|-----------|----------|
| School records | PostgreSQL | Indefinite | N/A |
| Person profiles | PostgreSQL | Indefinite | N/A |
| User credentials | PostgreSQL | Indefinite | N/A |
| User sessions | PostgreSQL | 90 days (configurable) | N/A |
| Profile images | S3 (object storage) | Duration of user account | 5 MB per image |
| Opaque session ids | PostgreSQL — row in `identity.user_session` (no JWT, no Redis session store) | Session TTL (configurable) | ~100 B per row |

### Authorization Service

| Asset type | Storage | Retention | Max size |
|-----------|---------|-----------|----------|
| Role definitions | PostgreSQL | Indefinite | N/A |
| Permission records | PostgreSQL | Indefinite | N/A |
| Role assignments | PostgreSQL | Indefinite | N/A |

### Academic Service

| Asset type | Storage | Retention | Max size |
|-----------|---------|-----------|----------|
| Student / Teacher records | PostgreSQL | Indefinite | N/A |
| Course and enrollment data | PostgreSQL | Indefinite | N/A |
| Academic documents (certificates, transcripts) | S3 (object storage) | 10 years after student graduation | 20 MB per document |
| Course syllabi | S3 (object storage) | Duration of course offering | 10 MB per file |

### Scheduling Service

| Asset type | Storage | Retention | Max size |
|-----------|---------|-----------|----------|
| Environment records | PostgreSQL | Indefinite | N/A |
| Schedule block records | PostgreSQL | Indefinite | N/A |
| Class session records | PostgreSQL | Indefinite | N/A |

### Attendance Service

| Asset type | Storage | Retention | Max size |
|-----------|---------|-----------|----------|
| Class session records | PostgreSQL | Indefinite | N/A |
| Attendance records | PostgreSQL | Indefinite | N/A |
| Justification documents (PDF, images) | S3 (object storage) | 5 years after period close | 10 MB per file |
| Attendance capture metadata | PostgreSQL | Indefinite | N/A |

### Biometric Service

| Asset type | Storage | Retention | Max size |
|-----------|---------|-----------|----------|
| Facial embedding vectors | MongoDB | Duration of user enrollment | 2 KB per vector |
| Fingerprint embedding vectors | MongoDB | Duration of user enrollment | 1 KB per vector |
| Facial update case images | S3 (object storage) | 30 days after review decision | 15 MB per image |
| Biometric model metadata | MongoDB | Duration of model version | 10 KB per record |

### Notification Service

| Asset type | Storage | Retention | Max size |
|-----------|---------|-----------|----------|
| Alert type catalog | PostgreSQL | Indefinite | N/A |
| Alert records | PostgreSQL | 2 years | N/A |

### Audit Service

| Asset type | Storage | Retention | Max size |
|-----------|---------|-----------|----------|
| Audit events | PostgreSQL | 7 years (compliance) | N/A |
| Error logs | PostgreSQL | 2 years | N/A |
| Audit event payloads (raw JSON) | S3 (object storage) | 3 years | 1 MB per record |

### Configuration Service

| Asset type | Storage | Retention | Max size |
|-----------|---------|-----------|----------|
| Academic configuration | PostgreSQL | Indefinite | N/A |
| Security configuration | PostgreSQL | Indefinite | N/A |
| Password policies | PostgreSQL | Indefinite | N/A |

---

## 3. File type summary

| Service | File types handled | Storage location |
|---------|-------------------|------------------|
| Identity | Profile images (JPEG, PNG) | S3 |
| Academic | PDFs, certificates, transcripts, syllabi | S3 |
| Attendance | Justification documents (PDF, JPEG, PNG) | S3 |
| Biometric | Facial images for review (JPEG, PNG) | S3 |
| Audit | Raw JSON payloads | S3 |

---

## 4. S3 bucket structure

```
s3://
└── faceattend-edu/
    ├── identity/           ← profile images
    │   └── {schoolId}/
    │       └── {personId}/
    ├── academic/           ← certificates, syllabi
    │   └── {schoolId}/
    │       └── {courseId}/
    ├── attendance/         ← justification documents
    │   └── {schoolId}/
    │       └── {periodId}/
    ├── biometric/          ← facial update case images
    │   └── {schoolId}/
    │       └── {caseId}/
    └── audit/              ← raw event payloads
        └── {year}/
            └── {month}/
```

---

## 5. Retention policy summary

| Retention period | Applies to |
|-----------------|------------|
| Indefinite | Core business records (all relational data) |
| 10 years | Academic documents (certificates, transcripts) |
| 7 years | Audit events (compliance) |
| 5 years | Attendance justification documents |
| 3 years | Audit raw payloads |
| 2 years | Error logs, Notification alerts |
| 90 days | User sessions |
| 30 days | Biometric review images after decision |
| Token TTL | Opaque session ids and auth sessions (minutes to days) |

---

## 6. Limits

| Limit | Value | Applies to |
|-------|-------|-----------|
| Max file size (profile image) | 5 MB | Identity |
| Max file size (academic document) | 20 MB | Academic |
| Max file size (justification) | 10 MB | Attendance |
| Max file size (biometric image) | 15 MB | Biometric |
| Max file size (audit payload) | 1 MB | Audit |

---

## Correlations

- Data ownership → `09-microservices/data-ownership-matrix.md`
- Service catalog → `09-microservices/service-catalog.md`
- Data model per service → `09-microservices/services/NN-name/data-model.md`
