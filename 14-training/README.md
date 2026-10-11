# Training & Onboarding - FaceAttendEDU

Welcome to the FaceAttendEDU technical team. This directory contains all the necessary resources to get a new developer up to speed, from the initial environment setup to the first production contribution.

Our onboarding process is designed to be hands-on, ensuring that every team member understands not only *how* to write code, but *why* we use specific architectural patterns and business rules.

---

## Onboarding Path

The onboarding process is divided into a structured 2-week plan to ensure a smooth transition:

### Week 1: Foundation & Context
- **Day 1: Setup & Access.** Environment configuration, tool access, and a high-level overview of the project.
- **Day 2: Domain Immersion.** Understanding the business logic, academic entities, and the "FaceAttendEDU" ecosystem.
- **Day 3: First Contribution.** Performing a "Good First Issue" using the TDD (Test Driven Development) workflow.
- **Days 4-5: Deep Dive.** Understanding the microservices architecture, quality standards, and participating in the first code reviews.

### Week 2: Guided Independence
- **Feature Implementation.** Taking ownership of a small user story.
- **API Mastery.** Understanding OpenAPI contracts and inter-service communication.
- **Integration.** Participating in sprint ceremonies and technical syncs.

---

## Key Documentation

To successfully complete the onboarding, the following documents are mandatory reading:

| Document | Purpose | Focus |
| :--- | :--- | :--- |
| `technical-onboarding.md` | **The Roadmap** | Step-by-step guide for the first 14 days. |
| `00-governance/git-conventions.md` | **The Rules** | Branching strategy and Conventional Commits. |
| `05-architecture/hexagonal-architecture.md` | **The Structure** | Domain-centric design and Ports & Adapters. |
| `11-quality/tdd-guide.md` | **The Quality** | How to implement the Red-Green-Refactor cycle. |

---

## Core Technical Concepts for Newcomers

Every developer at FaceAttendEDU must master these 5 pillars before working on the core domain:

1. **Bounded Contexts:** Each microservice (Identity, Attendance, Scheduling, etc.) represents a distinct part of the business.
2. **Hexagonal Architecture:** The domain is the heart. Frameworks (Spring Boot, React Native) are just adapters.
3. **Ports & Adapters:** We define interfaces in the domain and implement them in the infrastructure layer.
4. **Request Flow:** Controller $\rightarrow$ UseCase $\rightarrow$ Aggregate $\rightarrow$ Repository $\rightarrow$ Database.
5. **Domain Events:** Asynchronous communication between services to maintain eventual consistency.

---

## Quick Setup Checklist

- [ ] Local environment configured (See `10-devops/local-setup.md`).
- [ ] Docker containers for databases and API Gateway running.
- [ ] First `curl` to `/health` returns `{"status": "ok"}`.
- [ ] Git branch created following `feat/description` or `fix/description` format.

---

## Correlations
- **Technical Setup** $\rightarrow$ `10-devops/local-setup.md`
- **Coding Standards** $\rightarrow$ `00-governance/git-conventions.md`
- **Quality Strategy** $\rightarrow$ `11-quality/testing-strategy.md`
