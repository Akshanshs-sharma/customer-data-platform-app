## The Full Project — Checkpoint Map

Think of this as your **project roadmap**. Each checkpoint is a phase that produces a concrete deliverable.

```
PHASE 0 — Project Setup
PHASE 1 — Requirements Engineering
PHASE 2 — System Modeling
PHASE 3 — Architecture & Design
PHASE 4 — Data Modeling
PHASE 5 — Implementation
PHASE 6 — Testing
PHASE 7 — Documentation & Review
```

---

### PHASE 0 — Project Setup
**Goal:** Get the project infrastructure ready before any technical work.

Checkpoints:
- [ ] Create GitHub repository
- [ ] Define branching strategy (main, dev, feature branches)
- [ ] Set up issue labels (bug, feature, documentation, enhancement)
- [ ] Create project board (Kanban — Todo / In Progress / Done)
- [ ] Write a README skeleton

**Deliverable:** A live GitHub repo with structure ready to receive work.

---

### PHASE 1 — Requirements Engineering
**Goal:** Know exactly what we're building before touching design.

Checkpoints:
- [ ] Define actors (who uses the system)
- [ ] Write use cases / user scenarios per actor
- [ ] Finalize Functional Requirements (FR list)
- [ ] Finalize Non-Functional Requirements (NFR list)
- [ ] Requirements validation — check for gaps and conflicts
- [ ] Create GitHub Issues for each requirement

**Deliverable:** A `REQUIREMENTS.md` document in the repo.

---

### PHASE 2 — System Modeling
**Goal:** Visualize the system before designing it. (Sommerville Ch. 5)

Checkpoints:
- [ ] Context diagram — where does our CDP sit in the wider world?
- [ ] Use case diagram — actors and their interactions
- [ ] Sequence diagrams — key workflows step by step
- [ ] Class/domain model — high level entities

**Deliverable:** A `SYSTEM_MODEL.md` with diagrams.

---

### PHASE 3 — Architecture & Design
**Goal:** Decide how the system is structured technically.

Checkpoints:
- [ ] Choose architecture style (REST API, layered architecture)
- [ ] Define system components and their responsibilities
- [ ] Define API contracts (endpoints, request/response shapes)
- [ ] Technology stack decision with justification

**Deliverable:** An `ARCHITECTURE.md` document.

---

### PHASE 4 — Data Modeling
**Goal:** Design the database before writing any code.

Checkpoints:
- [ ] Identify all entities from requirements
- [ ] Draw ER diagram
- [ ] Write schema (table definitions, constraints, indexes)
- [ ] Validate schema against use cases

**Deliverable:** A `DATA_MODEL.md` with ER diagram and schema.

---

### PHASE 5 — Implementation
**Goal:** Build the actual system, feature by feature.

Checkpoints (one per FR):
- [ ] Project scaffolding (Spring Boot setup)
- [ ] Customer Profile CRUD
- [ ] Event Ingestion API
- [ ] Identity Resolution logic
- [ ] Trait Management
- [ ] Segment Definition & Evaluation

**Deliverable:** Working codebase on GitHub, each feature on its own branch merged via PR.

---

### PHASE 6 — Testing
**Goal:** Verify the system works correctly.

Checkpoints:
- [ ] Unit tests for business logic
- [ ] Integration tests for APIs
- [ ] Test documentation

**Deliverable:** Test suite in the repo, test report.

---

### PHASE 7 — Documentation & Review
**Goal:** Wrap it up professionally.

Checkpoints:
- [ ] Final README (setup, run, API guide)
- [ ] Retrospective — what did we learn?

**Deliverable:** Complete, presentable repository.

---
