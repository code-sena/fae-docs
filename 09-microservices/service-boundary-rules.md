# Service Boundary Rules

> Rules for deciding whether a capability belongs to one bounded context (service) or another. These rules apply to every domain entity and business operation in the platform.

---

## 1. Domain ownership

Each bounded context is the sole owner of its core entities. No other service may create, update, or delete rows that belong to a different context's canonical tables.

| Rule | Description |
|------|-------------|
| Single owner per entity | Every business entity has exactly one service that writes its canonical state |
| Context defines the model | The owning service defines the schema, validation rules, and lifecycle transitions |
| Cross-context reads are read-only | Other services may consume data through APIs or events, but never write directly |

---

## 2. Data lifecycle

An entity belongs to the context that **creates and maintains** it throughout its entire lifecycle.

| Lifecycle phase | Owner responsibility |
|-----------------|----------------------|
| Creation | The service that first instantiates the entity |
| Mutation | The service that enforces business rules on state transitions |
| Deletion / archival | The service that decides when and how an entity is removed or soft-deleted |
| Historical record | The service that retains audit trails for its own entities |

If a piece of data is created by one context but consumed by another, the consumer must hold a **local projection** (copy) keyed by the public ID — never a direct reference to the source database.

---

## 3. Local vs. external entities

| Scenario | Belongs to | Why |
|----------|-----------|-----|
| Entity is created, updated, and deleted within one context | Local | Full lifecycle ownership |
| Entity is referenced but never written by another context | External | Read-only reference via public ID |
| Entity is derived from another entity (e.g., academic profile from a person) | Derived context | The derived entity is a local projection, not a copy of the source |
| Entity is shared equally by two contexts with equal write rights | **Not allowed** | Redesign the boundary — one context must own it |

### Decision flowchart

```
Does this service CREATE the entity?
├─ YES → This service OWNS the entity
└─ NO
   Does this service UPDATE or DELETE the entity?
   ├─ YES → This service OWNS the entity
   └─ NO
      Does this service need to READ the entity?
      ├─ YES → Store a local projection keyed by public ID
      └─ NO → No relationship needed
```

---

## 4. Database ownership rule

> **If two things share a database, they are not independent services.**

| Principle | Implication |
|-----------|-------------|
| One database per bounded context | Each service connects to its own database instance or schema |
| No cross-service SQL joins | Joins across service boundaries are forbidden; use APIs or materialized views |
| No shared write tables | Two services must never INSERT/UPDATE the same table |
| Shared read tables are acceptable | A read-replica or projection table may be shared for query purposes, but the write owner is always one service |

When in doubt, ask: _"If this service's database went down, would the other service also break?"_ If the answer is yes, the boundary is wrong.

---

## 5. Cross-context references

References between bounded contexts use **public IDs** without foreign key constraints.

| Pattern | Example | Allowed |
|---------|---------|---------|
| Local FK | `Enrollment.student_id → Student.id` (same Academic context) | Yes |
| Public ID reference | `Attendance.student_id` references `Identity.User.user_id` without FK | Yes |
| Cross-service FK | `Attendance.student_id` FK to `Identity.User` table | **No** |
| Embedded aggregate | Storing the full person name inside attendance records | Acceptable for snapshots |

### Rules for public ID references

1. The referenced ID must be a UUID or stable public identifier, never an auto-increment local primary key.
2. The referencing service must not assume referential integrity — the referenced entity may not exist yet (eventual consistency).
3. When displaying cross-context data, the service must call the owning service's API or consume a projected view.

---

## 6. When to split a service

A new bounded context (and therefore a new service) should be created when:

| Signal | Action |
|--------|--------|
| Two sets of entities have different lifecycles | Split into separate contexts |
| Two teams own different parts of the data | Split to match team ownership |
| Scaling requirements differ (e.g., biometric vectors vs. relational data) | Split by storage need |
| A capability could be reused independently | Extract as its own context |

---

## 7. Quick reference — current service boundaries

| Service | Owns | Does NOT own |
|---------|------|--------------|
| Identity | School, Person, User, User_Session | Roles, enrollments, devices |
| Auth | Authentication sessions, JWT tokens | User credentials, person data |
| Authorization | Role, Permission, User_Role | User accounts, academic data |
| Academic | Student, Teacher, Course, Classroom, Period, Enrollment, Schedule | Person profiles, attendance |
| Attendance | Class_Session, Attendance, Justification | Biometric data, student records |
| Biometric | Facial_Embedding, Fingerprint_Embedding, Facial_Update_Case | Identity data, attendance records |
| IoT | IoT_Device, Device_Assignment | Classroom data, attendance |
| Configuration | Academic_Configuration, Security_Configuration, Password_Policy | Any core business entity data |

---

## Correlations

- Bounded context definitions → `02-domain/domain-map.md`
- Service catalog and ownership → `09-microservices/service-catalog.md`
- Data ownership matrix → `09-microservices/data-ownership-matrix.md`
