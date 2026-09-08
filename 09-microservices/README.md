# 09 — Microservices

> The platform is modeled as bounded contexts. Each context owns its own business data and can later become an independent service with its own database.
> Cross-context consistency is handled through public IDs and asynchronous events, never through direct foreign keys between contexts.

## Architectural rule applied to all services

The domain model establishes these constraints for the entire platform:

- FK only within the same bounded context.
- Between contexts, use public IDs without FK enforcement.
- Cross-context consistency is resolved through APIs and events.
- Each context may eventually evolve into a service with its own database.

This means the microservices architecture is not a pure monolith split by table, but a decomposition by business responsibility and data ownership.

---

## Bounded contexts in this project

| Context | Main responsibility | Main entities |
|---------|--------------------|---------------|
| Identity | school, people, users, sessions | School, Person, User, User_Session |
| Authorization | roles and access grants | Role, Permission, Role_Permission, User_Role |
| Academic | curricular and academic lifecycle | Student, Teacher, Course, Classroom, Period, Enrollment, Schedule |
| Attendance | real class sessions and attendance outcomes | Class_Session, Attendance, Justification_Type, Justification |
| Biometric | embeddings and recognition templates | Facial_Embedding, Fingerprint_Embedding, Facial_Update_Case |
| IoT | device lifecycle and classroom assignment | IoT_Device, Device_Assignment |
| Audit | operational traceability | Audit_Event, Error_Log |
| Configuration | technical and academic parameters | Academic_Configuration, Security_Configuration, Password_Policy |

---

## Service structure

Each service folder under `services/` follows the same structure:

```
services/
└── 01-service-name/
    ├── README.md              ← bounded context responsibility and rules
    ├── data-model.md          ← canonical model for that context
    ├── events.md              ← events published and consumed
    ├── decisions.md           ← service-specific design decisions
    ├── runbook.md             ← production operations
    └── components/            ← optional subcomponents
        └── [component]/
            ├── README.md
            └── contract.md
```

Use the template in `09-microservices/_template/service/` as the starting point for each new service.

---

## Cross-cutting documentation

### `service-catalog.md` ⭐
Registry of the current platform services and bounded contexts.

### `service-boundary-rules.md` ⭐
Rules for deciding whether a capability belongs to one service or another.
Key topics:
- domain ownership
- data lifecycle
- when an entity should remain local vs external
- database ownership rule: if two things share a DB, they are not independent services

### `communication-patterns.md` ⭐
How services communicate:
- synchronous APIs for queries and command orchestration
- asynchronous events for eventual consistency
- external ID references instead of direct DB joins

### `event-catalog.md` ⭐
List of the canonical business and operational events for the platform.

### `data-ownership-matrix.md` ⭐
Document who is the source of truth for each business entity.

### `dependency-map.md`
Map which service depends on which, ensuring there are no cycles in the domain relationships.

### `storage-and-documents.md`
Describe file, document, and media storage boundaries by service.

### `service-readiness-checklist.md`
Operational checklist before releasing a service.

---

## Correlations with other sections

| This section is fed by... | And feeds into... |
|---------------------------|-------------------|
| `02-domain/domain-map.md` → bounded contexts | one service per context |
| `05-architecture/overview.md` → defined microservice boundaries | README of each service |
| `07-api/contracts/openapi/` → service contracts | `services/[service]/` implementation |
| `06-data/models.md` → data model | `services/[service]/data-model.md` |
| `services/[service]/runbook.md` | `13-operations/` operational consolidation |

---

## Service numbering and grouping

The numbering reflects the domain dependency order rather than a strict package hierarchy:

- `01` = Identity
- `02` = Auth / Authorization
- `03` = Academic
- `04` = Attendance
- `05` = Biometric
- `06` = IoT
- `07` = Audit
- `08` = Configuration

Infrastructure pieces such as the API gateway remain support services and do not replace the domain-based bounded-context ownership model.
