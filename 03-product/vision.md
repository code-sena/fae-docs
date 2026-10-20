# Product Vision

> The vision is the team's north star. All sprints, design decisions,
> and trade-offs are evaluated against this vision.
> It must be ambitious yet achievable, inspiring but specific.

---

## Vision statement

**For** educational institutions, instructors, students, administrators, and supervisors  
**who** need to control attendance accurately, reduce manual roll call, prevent inconsistencies or impersonation, and track student participation in training environments,  
**the** FaceAttend EDU  
**is a** web and mobile attendance-management product based on facial recognition and reliable attendance records  
**that** automates entry and exit recording, centralizes attendance histories, supports justification workflows, and provides role-based reports and alerts,  
**unlike** manual attendance lists, paper records, or disconnected spreadsheets,  
**our product** combines facial recognition with manual and import registration methods, role-based visibility, justification traceability, biometric update workflows, and data-protection-oriented design for educational contexts such as SENA.

### Problem summary

Currently, attendance recording in many educational environments is performed manually or depends entirely on the instructor. This creates:

- Inconsistencies in attendance records.
- Greater administrative workload for the instructor.
- Difficulty tracking student participation in a timely manner.
- Risk of human error or impersonation.
- Difficulty consulting attendance, late-arrival, and absence histories.
- Lack of a structured process for justifying absences or late arrivals.

FaceAttend EDU addresses this problem through a technological solution that automates attendance recording, centralizes information, and facilitates report generation.

### Target users and value

| User | Main need | Value delivered |
|---|---|---|
| Student / Learner | Record attendance quickly, consult their history, and receive notifications related to their participation. | Fast, contactless recording, personal history visibility, and justification support. |
| Instructor / Teacher | View group attendance, receive absence alerts, and reduce the time spent taking attendance manually. | Less manual work, organized attendance information, and alerts for timely follow-up. |
| Administrator | Manage users, roles, permissions, environments, and general system configuration. | Centralized control of users, roles, academic structure, and system settings. |

---

## Team mission

We exist to modernize attendance management in educational environments by making attendance recording fast, reliable, transparent, and secure. Our purpose is to reduce instructors' administrative workload and help institutions detect and address participation issues early, so educational teams can focus more on training and less on manual control.

FaceAttend EDU should not be understood solely as a control tool, but as support for educational management, transparency, and institutional responsibility.

---

## Strategic pillars

Pillars are the focus areas that take us from mission to vision.  
They should be few and consistent over time.

| Pillar | Description | Success metrics |
|--------|-------------|----------------|
| Automated attendance capture | Reduce dependence on manual roll call by allowing students to record entry and exit through facial recognition or alternative methods. | Average recording time; percentage of successful facial records; percentage of alternative-method usage. |
| Reliable academic tracking | Provide accurate attendance, late-arrival, absence, and history information to instructors, administrators, and supervisors. | Record inconsistency rate; report consultation time; percentage of queries completed successfully by role. |
| Transparent justification workflow | Structure the process for uploading, evaluating, approving, or rejecting justifications for absences and late arrivals. | Percentage of justifications managed inside the system; average justification resolution time; percentage of automatic notifications delivered. |
| Security, privacy, and usability | Protect sensitive personal and biometric data while keeping the system easy to learn and use for users with basic ICT training. | Number of critical security incidents; user satisfaction score; usability/accessibility issues detected and resolved. |
| Scalable and compatible operation | Ensure the product can grow in users, cohorts, environments, and records while working correctly across modern browsers and compatible mobile devices. | Uptime; response time under load; number of compatibility issues by browser or device. |

---

## High-level roadmap

> Horizon 1: high certainty, core product validation. Horizon 2: traceability and adoption. Horizon 3: expansion and analytics.

```text
Phase 1 (0-3 months) ──── Phase 2 (3-6 months) ──── Phase 3 (6-12 months)
        │                         │                          │
      [MVP]                [Traceability]              [Scale / Expansion]
 Validate core             Justifications,             Fingerprint,
 attendance                dashboards, alerts,         advanced analytics,
 hypothesis                and config                  and multi-school
```

### MVP scope (Horizon 1) — what we ship first

| In scope | Out of scope (explicitly) |
|---|---|
| Identity + Authorization, Academic cohorts, Scheduling sessions, Attendance FACIAL/MANUAL/IMPORT, Justifications with docs, Biometric FACIAL enroll, Alerts (in-app), Reports by cohort/date | Fingerprint, IoT devices, Audit domain (now infra), payment/grading, external IdP, SMS/WhatsApp |

### Horizon 2 — Traceability

- Justification workflow end-to-end with approval SLAs, dashboards for instructors/admins, alert thresholds, configuration per school, mobile hardening.

### Horizon 3 — Scale

- Fingerprint biometrics via `Biometric_Update_Case`, advanced analytics, multi-school tenancy, Kafka replay for ops.
> Note 2026-10-15: fingerprint enrollment/verification already shipped in Web
> (`BiometricsView` + kiosk) ahead of this roadmap — treat as early delivery
> pending roadmap update, not as scope violation.

## Product principles (trade-offs)

1. **Accuracy over speed** — a missed attendance is worse than a 2s delay.
2. **Offline-first for capture** — queue and sync when connectivity returns (future).
3. **Privacy by design** — Law 1581 consent, encryption at rest, minimal PII in logs.

## Success criteria for next evaluation

- No placeholder text remains; all metrics have baseline + 6-month target + instrument.
- Domain product traces to SRS: RF 1-8 covered in pillars and roadmap.
- No language mixing (docs in English per `00-governance/documentation-rules.md`).