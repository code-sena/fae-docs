# 03 — Product Definition

> **What is this?**  
> The answer to “What are we going to build?”. It is not “how” — that belongs to architecture and implementation.  
> For **FaceAttend EDU**, this section defines the validated problem, the product vision, and the build plan for an attendance-management solution based on facial recognition, alternative recording methods, traceable histories, and justification workflows.

---

## Why this section exists

Without a clear product definition:

- The team may build features nobody asked for.
- Scope may grow out of control.
- There is no clear way to know whether the project was successful.
- Requirements from the SRS may be implemented without a shared product direction.

This section is the agreement between the team and stakeholders about **what will be built, why it will be built, and what will be excluded during the current project cycle**.

For FaceAttend EDU, this section prevents the team from prematurely building advanced analytics, complex IoT integrations, grading modules, payment features, or physical-access control before validating the core attendance problem.

---

## Product intent

This section translates the Software Requirements Specification (SRS) of FaceAttend EDU into product decisions.

The product definition is based on:

- The institutional problem described in the SRS: inefficient attendance validation and lack of effective tracking.
- The need to reduce instructors’ administrative workload.
- The need to improve attendance reliability and prevent inconsistencies.
- The need to provide traceable reports, justifications, and role-based visibility.
- The need to protect sensitive personal and biometric data.

---

## What is here and how to fill it in

### `problem-framing.md` ⭐ Start here

Articulates the problem before proposing solutions.

For FaceAttend EDU, this file should explain the attendance-control problem in educational environments before describing facial recognition as the solution.

**Suggested content from the SRS:**

```markdown
## The problem

**Who has it?**
Instructors, administrators, supervisors, students, and educational institutions, especially in contexts such as SENA training environments.

**What problem do they have?**
Attendance validation is deficient, inconsistent, and often depends on manual processes controlled by a single person. This creates administrative overload, errors, possible impersonation, and difficulty tracking student participation.

**When does it occur?**
At the beginning and end of training sessions, across different cohorts, environments, schedules, and instructors.

**What is the impact?**
Instructors lose time taking attendance manually. Institutions lose visibility into absences, late arrivals, and participation patterns. Students may face unclear attendance records. Administrative follow-up becomes harder.

**How do they solve it today?**
Typically through manual attendance lists, paper records, or informal instructor control. These methods are slow, error-prone, difficult to audit, and hard to consolidate into reports.

## Why it is worth solving

Automating attendance recording improves accuracy, reduces administrative workload, increases traceability, and allows instructors and institutions to focus more on training and less on manual control.