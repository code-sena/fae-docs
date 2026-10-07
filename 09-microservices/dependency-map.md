# Dependency Map

> Map of dependencies between services. The diagram and tables show which service depends on which, and ensure there are no cycles in domain-level relationships.

---

## 1. Dependency rules

| Rule | Description |
|------|-------------|
| No circular dependencies | If A depends on B, B must not depend on A at the domain level |
| Dependencies flow downward | Infrastructure depends on domain, never the reverse |
| Async breaks coupling | Event-based integration decouples runtime dependencies |
| Public IDs only | Cross-context references use UUIDs, not database-level FK constraints |

---

## 2. Dependency matrix

| Service | Depends on (sync API) | Depends on (async events) |
|---------|----------------------|---------------------------|
| API Gateway | Identity, Authorization, Academic, Scheduling, Attendance, Biometric, Notification, Configuration | — |
| Identity Service | — | Configuration (policy changes) |
| Authorization Service | — | Identity (UserCreated, SchoolCreated) |
| Academic Service | — | Identity (PersonCreated, SchoolCreated) |
| Scheduling Service | — | Academic (ScheduleBlockCreated) |
| Attendance Service | — | Academic (EnrollmentCreated), Scheduling (ClassSessionOpened), Biometric (BiometricVerified) |
| Biometric Service | — | Identity (UserCreated), Attendance (AttendanceRecorded) |
| Notification Service | — | Attendance (AttendanceRecorded, JustificationReviewed) |
| Configuration Service | — | Identity (SchoolCreated) |

---

## 3. Dependency diagram (ASCII)

```
                        ┌─────────────────┐
                        │   API Gateway    │
                        │  (Infrastructure)│
                        └────────┬────────┘
                                 │ sync
        ┌────────────┬───────────┼───────────┬────────────┬───────────┐
        ▼            ▼           ▼           ▼           ▼           ▼
  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
  │ Identity │ │Academic  │ │Scheduling│ │Attendance│ │Biometric │ │Notification│
  └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘
       │             │            │             │             │            │
       │             │            │             │             │            │
       │             │     ┌──────┘             │             │            │
       │             │     │                    │             │            │
       ▼             ▼     ▼                    ▼             ▼            ▼
  ┌──────────────────────────────────────────────────────────────────────┐
  │                    Event Bus (Kafka, KRaft :9092)                   │
  └──────────────────────────────────────────────────────────────────────┘
       │             │            │             │             │            │
       ▼             │            │             │             │            │
  ┌──────────┐       │            │             │             │            │
  │Config    │       │            │             │             │            │
  └──────────┘       │            │             │             │            │
                     │            │             │             │            │
                     └────────────┴─────────────┴─────────────┘            │
                                                                         │
                         (no domain-level cycles)                       │
                                                                         │
```

---

## 4. Layer diagram

```
┌─────────────────────────────────────────────────────────┐
│  INFRASTRUCTURE                                          │
│  API Gateway                                            │
└──────────────────────────┬──────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────┐
│  CORE DOMAIN SERVICES                                   │
│  Identity │ Authorization │ Academic │ Scheduling       │
│  Attendance │ Biometric                                   │
└──────────────────────────┬──────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────┐
│  SUPPORT SERVICES                                       │
│  Notification │ Configuration                           │
└─────────────────────────────────────────────────────────┘
```

---

## 5. Cycle detection

| Check | Result |
|-------|--------|
| Identity → Authorization → Identity | **No cycle** — Authorization consumes Identity events, but Identity does not depend on Authorization |
| Academic → Scheduling → Academic | **No cycle** — Scheduling consumes Academic events, but Academic does not depend on Scheduling |
| Academic → Attendance → Academic | **No cycle** — Attendance consumes Academic events, but Academic does not depend on Attendance |
| Scheduling → Attendance → Scheduling | **No cycle** — Attendance consumes Scheduling events, but Scheduling does not depend on Attendance |
| Biometric → Attendance → Biometric | **No cycle** — Biometric publishes events that Attendance consumes; Attendance publishes events that Biometric consumes, but these are independent event streams, not direct API calls |
| Configuration → Identity → Configuration | **No cycle** — Configuration consumes SchoolCreated; Identity consumes PasswordPolicyChanged. These are independent event flows, not circular dependencies |

> **Note:** Biometric ↔ Attendance is the closest to a bidirectional relationship. It is safe because both directions are asynchronous (event-driven), and neither service blocks on the other.

---

## 6. Service health in dependency chain

| If this service is down... | Impact |
|---------------------------|--------|
| Identity | Authorization, Academic, Biometric lose user resolution |
| Authorization | API Gateway cannot authorize requests; downstream services fail auth checks |
| Academic | Scheduling cannot validate cohorts; Attendance cannot validate enrollments |
| Scheduling | Attendance cannot validate class sessions |
| Attendance | Notification loses alert triggers |
| Biometric | Attendance cannot correlate biometric captures |
| Notification | No immediate operational impact; alerts queue in DLQ |
| Configuration | Identity and Academic may operate with stale policies |

---

## Correlations

- Service catalog → `09-microservices/service-catalog.md`
- Communication patterns → `09-microservices/communication-patterns.md`
- Bounded context definitions → `02-domain/domain-map.md`
