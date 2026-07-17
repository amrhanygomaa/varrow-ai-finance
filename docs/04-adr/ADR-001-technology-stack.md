# ADR-001: Technology Stack Selection

| | |
|---|---|
| **ADR ID** | ADR-001 |
| **Title** | Technology Stack Selection |
| **Status** | Accepted |
| **Author** | Architecture Team |
| **Date** | 2026-07-17 |
| **Project** | VArrow AI Finance |
| **Company** | VArrow.tech |

---

## Context

VArrow AI Finance is an AI-powered Financial Operations Platform for a single company (internal deployment, not SaaS), delivered on Web and Flutter Mobile, with AWS as the primary cloud and Google Cloud AI (Gemini) as a secondary AI service. This ADR records the **final, accepted** technology stack decisions for the project and provides the reasoning, trade-offs, and consequences behind each.

These decisions are **final**. This document exists to justify and preserve the rationale, and to establish the baseline against which future ADRs may propose changes.

> This ADR is a decision record only. It does not create infrastructure, provision cloud resources, or produce implementation.

---

## Backend

**Decision: Node.js + NestJS + TypeScript**

### Reasoning
- **NestJS** provides an opinionated, modular architecture that aligns naturally with the platform's modular design, offering a clear structure for modules, services, and boundaries out of the box.
- **TypeScript** brings static typing and strong tooling, which materially reduces defects in a financial domain where correctness and maintainability are paramount.
- **Node.js** offers a mature ecosystem, strong asynchronous I/O suited to API-driven financial operations, and a large talent pool.
- A single language (TypeScript) shared with the frontend reduces cognitive load, enables shared types and validation logic, and streamlines full-stack development for a startup-sized team.

### Trade-offs
- Node.js is single-threaded per process for CPU-bound work; heavy computation must be handled carefully (offloading, batching) rather than blocking the event loop.
- NestJS's abstraction and dependency-injection patterns add a learning curve for developers new to the framework.
- The npm ecosystem requires disciplined dependency management to control supply-chain and security risk.

### Alternatives Considered
- **Java / Spring Boot** — very strong in enterprise finance, but heavier, slower to iterate for a startup, and introduces a second language away from the TypeScript frontend.
- **Python / FastAPI** — excellent for AI-adjacent work and rapid development, but weaker language-level typing guarantees than TypeScript and a language split with the frontend.
- **.NET / C#** — robust and enterprise-grade, but a heavier operational and licensing footprint and, again, a language divergence from the frontend.
- **Plain Node.js (Express/Fastify)** — lighter, but lacks the built-in modular structure and conventions NestJS provides, pushing that structure onto the team to define and maintain.

### Decision
Adopt **Node.js with NestJS and TypeScript** as the backend stack.

### Consequences
- **Positive:** Consistent structure across modules, shared language and types with the frontend, strong typing in a financial domain, and productive full-stack development.
- **Positive:** NestJS conventions ease onboarding once learned and support the modular monolith style chosen below.
- **Negative:** The team must observe good practices for CPU-bound tasks and dependency hygiene.
- **Negative:** Framework conventions must be respected consistently to avoid architectural drift.

---

## Frontend

**Decision: React + TypeScript + Vite**

### Reasoning
- **React** is a mature, widely adopted library with a deep ecosystem and a large talent pool, reducing hiring and maintenance risk.
- **TypeScript** aligns the frontend with the backend language, enabling shared types and a consistent developer experience across the stack.
- **Vite** provides fast development startup, rapid hot-module reloading, and an efficient build pipeline, improving developer productivity.
- The combination supports building a rich, responsive web experience for financial operations with bilingual (English/Arabic, including right-to-left) requirements.

### Trade-offs
- React is a library rather than a full framework, so some architectural choices (routing, state management, data fetching) must be made and standardized by the team.
- The React ecosystem evolves quickly, requiring ongoing attention to keep dependencies current.
- Right-to-left and mixed-language support must be handled deliberately in the frontend design.

---

## Mobile

**Decision: Flutter**

### Reasoning
- **Flutter** delivers a single codebase targeting both major mobile platforms, which is efficient for a startup delivering a companion mobile experience.
- It provides a consistent, high-quality UI across platforms and strong support for custom, branded interfaces.
- Flutter has mature support for localization, including right-to-left layouts required for Arabic.

### Trade-offs
- Flutter uses **Dart**, introducing a language distinct from the TypeScript used across web and backend, which adds a second skill set for the team.
- Sharing logic between the Flutter mobile app and the TypeScript stack is limited, so some concerns (e.g., validation rules) may be expressed in more than one place.
- Native platform-specific capabilities may occasionally require platform channels or additional effort.

---

## Database

**Decision: PostgreSQL**

### Reasoning
- **PostgreSQL** is a mature, reliable, ACID-compliant relational database, well suited to the transactional integrity requirements of financial operations.
- It offers strong support for complex queries, constraints, and data integrity, which are essential for invoices, payments, approvals, and reporting.
- It is well supported on AWS and within the Node.js/NestJS ecosystem, and it handles both structured relational data and flexible JSON-based data when needed.

### Trade-offs
- As a primarily relational system, very high write-scale or specialized workloads may eventually require complementary data stores; this is acceptable for a single-company deployment.
- Schema evolution requires disciplined migration practices to protect financial data integrity over time.

---

## Cloud

**Decision: AWS (Primary Cloud)**

### Reasoning
- **AWS** is the designated primary cloud for the platform, offering a broad, mature set of managed services that reduce operational burden for a startup.
- It provides strong reliability, security, and compliance capabilities appropriate for handling sensitive financial data.
- Its managed database, compute, storage, and AI services (including AWS Bedrock, see below) align well with the chosen stack.

### Trade-offs
- Reliance on AWS introduces a degree of cloud-provider dependency; this is mitigated by favoring portable technologies (Node.js, PostgreSQL, containers) where practical.
- Cost management requires ongoing attention to avoid unexpected spend as usage grows.
- Using Google Gemini as the primary AI service alongside AWS introduces a cross-cloud dependency that must be managed operationally (see AI section).

---

## AI

**Decision: Primary — Google Gemini; Secondary — AWS Bedrock**

### Reasoning
- **Google Gemini** is selected as the **primary** AI service for its capabilities in language understanding and generation, including support relevant to bilingual (English/Arabic) and mixed-language financial documents.
- **AWS Bedrock** is selected as the **secondary** AI service, providing an alternative within the primary cloud (AWS) and access to a range of foundation models. This offers resilience, flexibility, and alignment with the primary cloud.
- A primary/secondary arrangement reduces single-provider dependency for AI, allowing fallback and workload-appropriate routing while keeping humans in control of financial decisions.

### Trade-offs
- Using a primary AI provider (Google) outside the primary cloud (AWS) introduces cross-cloud operational and data-governance considerations that must be managed carefully.
- Maintaining two AI providers increases integration and governance overhead compared to a single provider.
- Provider capabilities and terms evolve; the arrangement must be periodically reviewed to remain appropriate.

---

## Deployment

**Decision: Single Company (Internal Platform, not SaaS)**

### Reasoning
- The platform is deployed for **one company's internal use**, not as a multi-tenant SaaS product. This simplifies the architecture by removing multi-tenancy concerns such as tenant isolation, per-tenant configuration, and tenant-aware billing.
- A single-company scope allows the team to focus on depth of financial operations functionality and AI assistance rather than the complexity of a multi-tenant platform.
- It aligns with the product vision, which explicitly positions the platform as a single-company solution and not an ERP replacement.

---

## Architecture Style

**Decision: Modular Monolith**

### Reasoning
- A **modular monolith** provides clear internal module boundaries (aligned with the business module catalog) while retaining the operational simplicity of a single deployable application.
- For a startup delivering a single-company platform, this maximizes development velocity and minimizes operational overhead compared to a distributed microservices architecture.
- NestJS's modular structure directly supports this style, encouraging well-defined modules that can be developed and reasoned about independently within one codebase.

### Trade-offs
- A monolith scales as a single unit; independent scaling of individual modules is limited compared to microservices.
- Strong internal discipline is required to keep module boundaries clean and avoid tight coupling that would make a future split harder.
- A single deployable means a fault in one area can affect the whole application, requiring good testing and resilience practices.

### Future Migration Path to Microservices
- The modular monolith is deliberately structured so that **well-defined module boundaries** can later become service boundaries if scale or organizational needs demand it.
- Should specific modules require independent scaling, deployment, or ownership, they can be **extracted incrementally** into separate services, starting with the most independent and highest-value candidates.
- Maintaining clear contracts between modules, avoiding shared internal state across boundaries, and keeping cross-module interactions explicit will preserve this option without committing to microservices prematurely.
- This path is a **future option, not a current commitment**; the platform begins as a modular monolith and migrates only if and when justified.

---

## Document Status

| | |
|---|---|
| **Status** | Accepted |
| **Author** | Architecture Team |
| **Date** | 2026-07-17 |

---

## Summary of Decisions

| Layer | Decision |
|---|---|
| Backend | Node.js + NestJS + TypeScript |
| Frontend | React + TypeScript + Vite |
| Mobile | Flutter |
| Database | PostgreSQL |
| Cloud | AWS (Primary) |
| AI — Primary | Google Gemini |
| AI — Secondary | AWS Bedrock |
| Deployment | Single Company (not SaaS) |
| Architecture Style | Modular Monolith (with future microservices migration path) |

---

*This ADR records final, accepted decisions and their rationale. It does not create infrastructure or implementation. Future changes to these decisions must be captured in subsequent ADRs that supersede the relevant sections of this record.*
