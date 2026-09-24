# Normalization Assessment — FaceAttend EDU

> **Corte:** 2026-09-22 · **Ficha:** ADSO-3145556 · **Repo:** `code-sena/fae-docs`
> **Alcance:** 29 entidades de `02-domain/entities-and-rules.md` frente a 31 tablas SQL (PostgreSQL 17) + 2 colecciones MongoDB documentadas en `06-data/domains/`
> **Veredicto global:** **3NF sin excepciones — sin denormalizaciones intencionales.** Cada tabla está en 3NF; no hay atributos transitivos ni grupos repetidos. Lo que parece denormalización (FK externas sin constraint físico, `current_embedding_ref` SQL→NoSQL) es aislamiento de bounded context, no duplicación.

---

## 1. Método

- Fuente primaria: `06-data/domains/0*.md` (DDL SQL + diccionarios) y `02-domain/entities-and-rules.md` / `domain-map.md`.
- Se verificó 1NF → 2NF → 3NF tabla por tabla con el criterio clásico:
  - **1NF:** dominio atómico por columna, sin listas ni valores compuestos, PK definida.
  - **2NF:** si PK compuesta → ningún atributo no-clave depende de una parte de la PK.
  - **3NF:** ningún atributo no-clave depende transitivamente de otro no-clave.
- BCNF se revisa solo donde hay dependencias funcionales no triviales sobre superclave (ver §5).
- Actividad y trazabilidad contrastadas contra `04-requirements` (39 HU) y `09-microservices/service-catalog.md`.

---

## 2. Resumen ejecutivo

| Bounded Context | Entidades (02-domain) | Tablas SQL / Colecciones | Veredicto |
|---|---:|---|---|
| Identity | 3 (Person, App_User, City) + VOs | 5 tablas (`city`, `person`, `app_user`, `user_session`, `password_policy`) | 3NF |
| Authorization | 4 (Role, Permission, Role_Permission, User_Role) | 4 tablas | 3NF |
| Academic | 8 (School, Program, Academic_Period, Cohort, Course, Academic_Actor_Type, Academic_Actor, Enrollment) | 8 tablas | 3NF |
| Scheduling | 3 (Environment, Schedule_Block, Class_Session) | 3 tablas | 3NF |
| Attendance | 4 (Attendance_Record, Justification_Type, Justification, Supporting_Document) | 4 tablas | 3NF |
| Biometric | 2 (Facial_Embedding, Fingerprint_Embedding) | 2 colecciones MongoDB | N/A relacional — documento normalizado por diseño* |
| Notification | 2 (Alert_Type, Alert) | 2 tablas | 3NF |
| Audit | 2 (Audit_Log, Error_Log) | 2 tablas | 3NF (append-only) |
| Configuration | 3 (Academic_Configuration, Security_Configuration, Biometric_Update_Case) | 3 tablas | 3NF |
| **Total** | **29 entidades + 4 VOs** | **31 SQL + 2 Mongo** | **3NF, 0 denormalizaciones** |

* Biométrico no se evalúa con formas normales relacionales; se evalúa como documento con referencias externas por UUID/ObjectId (ver §7).

**Conteo histórico (informe 2026-09-22):** el informe reportó "30 tablas". La diferencia es `password_policy` (singleton global en Identity). Contando solo tablas de negocio multi-fila son 30; incluyendo el singleton son 31. Ambas cifras son correctas según criterio — aquí se documentan las 31.

> Entidad↔tabla: calce 29→29 perfecto salvo `Facial_Embedding`/`Fingerprint_Embedding` que deliberadamente **no** tienen tabla SQL — viven en MongoDB (`06-data/domains/06-biometric.md`). No es hueco: es decisión de motor documentada en ADR-003.

---

## 3. 1NF — Atomicidad

**Veredicto: cumple en todas las tablas.**

- Ninguna columna almacena listas, CSV, JSON sin justificar, ni valores compuestos.
- Tiempo modelado correctamente: `schedule_block.starts_at`/`ends_at` son `TIME` atómicos; `academic_period.starts_on/ends_on` son `DATE`; `attendance_record.captured_at` es `TIMESTAMPTZ`.
- Ejemplo positivo citado: `enrollment` no guarda `student_names[]`; `schedule_block` no guarda `days VARCHAR "Mon,Wed,Fri"` — usa una fila por bloque con `day_of_week SMALLINT` + constraints de unicidad.
- Contrapunto 1NF clásico evitado: se separó `person.document_number` de `person.name/last_name` (no "full_name"), y `academic_actor.actor_code` separado de `school_id`.

---

## 4. 2NF — Sin dependencias parciales

**Veredicto: cumple.**

Solo 2 tablas tienen PK compuesta y ambas están correctamente en 2NF:

| Tabla | PK | Atributos no-clave | ¿Depende de PK completa? |
|---|---|---|---|
| `role_permission` | `(role_id, permission_id)` | `assignment_date` + audit | Sí — la fecha pertenece a la asignación, no al rol ni al permiso aislados |
| `user_role` | `(user_id, role_id)` | `assignment_date` + audit | Sí — misma lógica; no hay atributo que describa solo a `user_id` o solo a `role_id` |

Todas las demás tablas tienen PK simple (SERIAL/BIGSERIAL/UUID) → 2NF es automática si 1NF se cumple.

---

## 5. 3NF y BCNF — Sin dependencias transitivas

**Veredicto: 3NF en las 31 tablas. BCNF también se cumple donde aplica.**

Revisión por patrón:

### 5.1 Tablas sin riesgo transitivo (atributos dependen solo de la PK)

`city`, `role`, `permission`, `school`, `program`, `academic_period`, `cohort`, `course`, `academic_actor_type`, `environment`, `class_session`, `justification_type`, `supporting_document`, `alert_type`, `alert`, `audit_log`, `error_log`, `academic_configuration`, `security_configuration` — todos sus atributos son hechos de la PK, no de otro no-clave. No hay `school_name` duplicado en `program`, ni `city_name` en `school`; se referencia por `school_id`/`city_id`.

### 5.2 Tablas con FKs que podrían confundirse con transitividad

| Tabla | FKs | ¿Transitiva? | Justificación |
|---|---|---|---|
| `academic_actor` | `actor_type_id → academic_actor_type`, `school_id → school` | No | `actor_type_id` y `school_id` son FKs a entidades distintas; ningún atributo de `academic_actor` depende de `actor_type.name` ni de `school.name` |
| `enrollment` | `academic_actor_id`, `cohort_id` | No | `enrollment_status` depende de la matrícula `(actor,cohort)`, no del actor ni del cohort aislados |
| `schedule_block` | `cohort_id`, `course_id`, `environment_id`, `instructor_actor_id` | No | Cada FK es identificador externo; horarios dependen del bloque, no del nombre del curso |
| `attendance_record` | `class_session_id` (ext), `academic_actor_id` (ext) | No | `attendance_status`, `capture_method`, `match_score` dependen del registro, no de la sesión |
| `justification` | `attendance_record_id` (UK), `justification_type_id` | No | `reason`, `review_status` dependen de la justificación, no del tipo |
| `biometric_update_case` | `person_id` (ext), `current_embedding_ref` (Mongo logical) | No | `biometric_type`, `update_status` dependen del caso |

### 5.3 Casos que *parecen* denormalización y no lo son

- **`alert.home_id`-análogo:** aquí `academic_configuration.school_id` y `scheduling.environment.school_id` son **FK cross-context sin constraint físico** (externas por UUID/SERIAL). No reforzar la FK a nivel de motor es una decisión de aislamiento de microservicios (Database per Service), no una denormalización — no duplica datos.
- **`city_id` en `school`:** almacenar solo el ID y no el nombre de la ciudad es precisamente la forma normalizada; duplicar `city_name` en `school` sí sería denormalización (y no se hace).
- **FK nulable ≠ denormalización:** `class_session.opened_by/closed_by` y `biometric_update_case.reviewed_by` pueden ser NULL antes de la acción. Eso es relación opcional, no dato duplicado.

### 5.4 BCNF

Se evaluó BCNF en tablas con dependencias funcionales no triviales: `enrollment (academic_actor_id, cohort_id) → enrollment_id` no aplica porque la PK es surrogate; la DF real es `enrollment_id → (actor, cohort)` y `enrollment_id` es superclave → BCNF. `academic_actor (school_id, actor_type_id, actor_code)` tiene UNIQUE sobre esa terna, pero la PK sigue siendo `academic_actor_id` (superclave) → BCNF.

---

## 6. Matriz tabla por tabla (veredicto 3NF)

| # | Tabla / Colección | PK | 1NF | 2NF | 3NF | Notas |
|---|---|---|---|---|---|---|
| 1 | `city` | `city_id` | ✓ | ✓ | ✓ | Catálogo simple |
| 2 | `person` | `person_id` UUID | ✓ | ✓ | ✓ | `document_number` UK, sin `school_id` (multi-tenant vía actor) |
| 3 | `app_user` | `user_id` UUID | ✓ | ✓ | ✓ | 1:1 con person, `username` UK |
| 4 | `user_session` | `session_id` UUID | ✓ | ✓ | ✓ | Sesión por usuario |
| 5 | `password_policy` | `policy_id` SERIAL | ✓ | ✓ | ✓ | Singleton global — 1 fila efectiva |
| 6 | `role` | `role_id` SERIAL | ✓ | ✓ | ✓ | Global, no `school_id` (decisión v4) |
| 7 | `permission` | `permission_id` SERIAL | ✓ | ✓ | ✓ |  |
| 8 | `role_permission` | `(role_id, permission_id)` | ✓ | ✓ | ✓ | M:N con `assignment_date` |
| 9 | `user_role` | `(user_id, role_id)` | ✓ | ✓ | ✓ | `user_id` externo, sin FK física |
| 10 | `school` | `school_id` SERIAL | ✓ | ✓ | ✓ | `code` UK, `city_id` externo |
| 11 | `program` | `program_id` SERIAL | ✓ | ✓ | ✓ | `uq_program_code_school` |
| 12 | `academic_period` | `academic_period_id` SERIAL | ✓ | ✓ | ✓ | `uq_period_name_school` |
| 13 | `cohort` | `cohort_id` BIGSERIAL | ✓ | ✓ | ✓ | `code` UK |
| 14 | `course` | `course_id` SERIAL | ✓ | ✓ | ✓ | `uq_course_code_program` |
| 15 | `academic_actor_type` | `actor_type_id` SMALLINT | ✓ | ✓ | ✓ | `STUDENT`/`INSTRUCTOR` |
| 16 | `academic_actor` | `academic_actor_id` BIGSERIAL | ✓ | ✓ | ✓ | 2 UK: `(school,type,code)` y `(person,type,school)` |
| 17 | `enrollment` | `enrollment_id` BIGSERIAL | ✓ | ✓ | ✓ | UNIQUE `(actor,cohort)` |
| 18 | `environment` | `environment_id` SERIAL | ✓ | ✓ | ✓ | `uq_environment_code_school` |
| 19 | `schedule_block` | `schedule_block_id` BIGSERIAL | ✓ | ✓ | ✓ | 2 UNIQUE anti-doble-reserva (room, instructor) |
| 20 | `class_session` | `class_session_id` BIGSERIAL | ✓ | ✓ | ✓ | UNIQUE `(block, date)` |
| 21 | `attendance_record` | `attendance_record_id` BIGSERIAL | ✓ | ✓ | ✓ | UNIQUE `(session, actor)` |
| 22 | `justification_type` | `justification_type_id` SERIAL | ✓ | ✓ | ✓ | `requires_attachment` |
| 23 | `justification` | `justification_id` BIGSERIAL | ✓ | ✓ | ✓ | UNIQUE `attendance_record_id` (1:1) |
| 24 | `supporting_document` | `supporting_document_id` BIGSERIAL | ✓ | ✓ | ✓ | FK → justification |
| 25 | `alert_type` | `alert_type_id` SMALLINT | ✓ | ✓ | ✓ | `code` UK |
| 26 | `alert` | `alert_id` BIGSERIAL | ✓ | ✓ | ✓ | `resolved_at` NULL = abierto |
| 27 | `audit_log` | `audit_log_id` BIGSERIAL | ✓ | ✓ | ✓ | Append-only |
| 28 | `error_log` | `error_id` BIGSERIAL | ✓ | ✓ | ✓ | Append-only |
| 29 | `academic_configuration` | `configuration_id` SERIAL | ✓ | ✓ | ✓ | UNIQUE `(school, name)` |
| 30 | `security_configuration` | `configuration_id` SERIAL | ✓ | ✓ | ✓ | `configuration_name` UK global |
| 31 | `biometric_update_case` | `case_id` UUID | ✓ | ✓ | ✓ | `biometric_type` + `finger_number` condicional |
| 32 | `facial_embeddings` (Mongo) | `_id` ObjectId | — | — | — | Vector `encoding`, `is_active`, `person_id` ext |
| 33 | `fingerprint_embeddings` (Mongo) | `_id` ObjectId | — | — | — | `finger_number` 1..10, `is_active` |

---

## 7. Denormalizaciones intencionales

**Ninguna.**

> "As of this writing, the schema has no intentional denormalization — every table is in 3NF. This is stated plainly rather than manufactured content to fill the section." — mismo criterio que `energy-monitor` (reportado como el más limpio del corte).

Si en el futuro se introduce una denormalización (ej. cachear `school_code` en `enrollment` para evitar JOIN, o `actor_code` en `alert`), deberá documentarse aquí con: (a) tabla/columna duplicada, (b) motivo (performance vs JOIN), (c) mecanismo de honestidad (trigger, job de reconciliación, o `updated_at` de fuente), y (d) costo de escritura.

---

## 8. Colecciones NoSQL (Biometric)

`facial_embeddings` y `fingerprint_embeddings` no se normalizan con 1NF-3NF relacional. Su diseño equivalente a "normalizado" es:

- Un documento por embedding, sin arrays anidados que violen atomicidad.
- Índices ` (person_id, is_active)` y `(person_id, finger_number, is_active)` garantizan "máximo un activo por persona/dedo" a nivel de aplicación (RN-28), no por constraint único Mongo — documentado como invariante de dominio.
- `person_id` es referencia externa a `Identity.person` — sin DBRef ni FK, resuelta por eventos/API.
- `biometric_update_case.current_embedding_ref` (SQL) referencia `_id` (Mongo) como `VARCHAR` lógico — cross-paradigm, sin FK.

---

## 9. Trazabilidad y auditoría

- Campos de auditoría estándar en **todas** las tablas SQL: `created_at`, `updated_at`, `deleted_at` (soft delete), `created_by`, `updated_by`, `deleted_by`, `row_version` (optimistic locking) — definidos en `06-data/models.md` y replicados en cada DDL.
- `audit_log`/`error_log` son append-only (RN-44): nunca UPDATE/DELETE.
- Índices de borrado lógico: `WHERE deleted_at IS NULL` en tablas de alto volumen (`person`, `academic_actor`, `enrollment`, `attendance_record`).

---

## 10. Open items

| ID | Descripción | Estado |
|---|---|---|
| DATA-01 | Verificar con carga real que `uq_actor_person_type_school` no bloquea el caso "misma persona como STUDENT e INSTRUCTOR en dos schools distintas" | Pendiente de test de integración |
| DATA-02 | Decidir si `password_policy` migra a `configuration-service` en v5 (hoy vive en Identity por simplicidad) | Discusión abierta — no afecta normalización |

---

## 11. Conclusión

El modelo FaceAttend EDU está en **3NF sin excepciones** con 31 tablas SQL + 2 colecciones Mongo, trazable 1:1 a 29 entidades + 4 VOs de `02-domain/entities-and-rules.md`. No hay deuda de normalización pendiente. La única "divergencia" señalada por el detector automático (`Facial_Embedding` sin tabla SQL) es intencional y correcta por diseño (MongoDB).

---

## Correlaciones

- Entidades y reglas → `02-domain/entities-and-rules.md`
- Mapa de contextos → `02-domain/domain-map.md`
- Modelo por dominio (DDL) → `06-data/domains/0*.md`
- Resumen transversal → `06-data/models.md`
- Patrones Saga/Outbox → `05-architecture/pattern-guide.md`
- Contratos API → `07-api/contracts/openapi/`
- Catálogo de servicios y ownership → `09-microservices/service-catalog.md`, `09-microservices/data-ownership-matrix.md`
