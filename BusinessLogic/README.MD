# BusinessLogic Knowledge Base

## Purpose

This directory contains the **authoritative description of Modula’s business behavior**.

It is the source of truth for:
- Business rules and invariants
- Domain models
- Cross-module business workflows
- Module-level specifications used for implementation

Everything in this directory describes **what the system is and does**, not *how the documentation is written*.

If a behavior is implemented in code, it must be traceable to a document here.

---

## What This Directory Is (and Is Not)

### This directory **IS**:
- A shared knowledge base for the team
- A reference for developers implementing Modula
- A foundation for academic artifacts (domain modeling, requirement engineering)
- A contract between business understanding and software implementation

### This directory **IS NOT**:
- A collection of documentation guidelines or methodologies
- A place for “how to write docs” instructions
- A substitute for infrastructure, deployment, or operational playbooks

Documentation **methodology** lives outside this directory.

---

## Relationship to Playbooks and Templates

- **Playbooks** define *how* we model and document the system  
- **Templates** define *how documents are structured*
- **BusinessLogic** contains the *actual modeled knowledge*

Refer to:
- `KnowledgeBase/Playbooks/BusinessLogic_Modeling_Playbook.md`
- `KnowledgeBase/Templates/*`

for documentation standards and modeling rules.

---

## Directory Structure
```
BusinessLogic
├── README.md
│
├── domain
│   └── SelfContained
│       ├── inventory_domain.md
│       ├── cashSession_domain.md
│       ├── order_domain.md
│       └── menu_domain.md
│
├── process
│   └── CrossModule
│       ├── finalizeOrder_process.md
│       ├── voidOrder_process.md
│       ├── endOfDayClose_process.md
│       └── syncReplay_process.md
│
└── modSpec
    ├── SelfContained
    │   ├── inventory_module.md
    │   ├── cashSession_module.md
    │   └── …
    └── CrossModuleProcess
        ├── finalizeOrder_modspec.md
        ├── voidOrder_modspec.md
        └── …
```

---

## How to Read This Knowledge Base

### 1. Domain Models (`domain/SelfContained`)
These documents answer:

> **“What rules must always be true?”**

They define:
- Domain boundaries and ownership
- Aggregates and entities
- Invariants and lifecycle rules
- Commands accepted and events emitted
- Explicit out-of-scope responsibilities

Domains are **self-contained** and do **not** embed cross-module orchestration.

---

### 2. Cross-Module Processes (`process/CrossModule`)
These documents answer:

> **“How do multiple domains work together to fulfill a business operation?”**

They define:
- Orchestrated workflows across domains
- Ordering of steps
- Transaction boundaries
- Idempotency and retry strategy
- Failure modes and compensations

Process documents **do not own persistent state**.

---

### 3. Module Specifications (`modSpec`)
These documents answer:

> **“How is this implemented?”**

They translate domain rules and processes into:
- Use cases
- APIs and data models
- Acceptance criteria
- Implementation constraints

Module specs are split into:
- **Self-contained module behavior**
- **Cross-module process participation**

---

## Core Architectural Rules

- Domains **must not** contain cross-module orchestration logic
- Cross-domain business logic **must** be documented as a process
- Module specs **must** reflect the domain and process documents
- Append-only, idempotent, and auditable behavior is preferred where applicable
- Corrections are modeled explicitly (e.g., adjustments), not via mutation

---

## Scope and Evolution

This knowledge base evolves intentionally.

- **March Delivery / Capstone I**
  - Café-grade POS and inventory behavior
  - Offline-first, audit-friendly workflows
  - Explicitly scoped, non-ERP-level complexity

- **Future Phases**
  - Advanced inventory (e.g. FEFO enforcement)
  - Multi-branch logistics
  - Extended reporting and analytics

Future behavior must be introduced by:
1. Updating the relevant **domain model**
2. Adding or updating **process documentation**
3. Patching affected **module specifications**

---

## Final Note

If a rule, workflow, or invariant is **not documented here**, it is **not considered part of the system’s business logic**.

This directory is the **single source of truth for Modula’s business behavior**.