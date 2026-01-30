# BusinessLogic Knowledge Base

## Purpose

This directory is the **authoritative knowledge base for Modula’s business behavior**.

It defines:
- **What the system is**
- **What rules must always be true**
- **How different parts of the system cooperate**
- **What developers (human or AI) are allowed to implement**

If a behavior exists in code, it **must be traceable** to a document here.

This directory is intentionally written to be:
- understandable by developers
- defensible in academic review
- useful for long-term system evolution

---

## What This Directory Is (and Is Not)

### This directory **IS**:
- The single source of truth for business rules and workflows
- A shared mental model for the team
- A bridge between product thinking and implementation
- A reference for backend AI and frontend AI development

### This directory **IS NOT**:
- Infrastructure documentation
- Deployment or DevOps guides
- Coding style or framework instructions
- Sprint tickets or task breakdowns

Those belong elsewhere.

---

## High-Level Structure

````
BusinessLogic
├── README.md
│
├── domain
│   ├── 30_POSOperation
│   │   ├── order_domain_patched.md
│   │   ├── cashSession_domain.md
│   │   ├── inventory_domain.md
│   │   ├── menu_domain_patched_v2.md
│   │   └── discount_domain.md
│   │
│   └── _archived
│       └── historical / superseded domain drafts
│
├── process
│   ├── finalizeOrder_process.md
│   ├── stock_deduction_on_finalize_sale_process.md
│   ├── void_sale_inventory_reversal_process.md
│   ├── voidOrder_process.md
│   └── deviceDraft_process.md
│
├── modSpec
│   ├── 00_guide
│   │   └── version.md
│   │
│   ├── 10_IdentityAccess
│   ├── 20_OrgAccount
│   ├── 30_POSOperation
│   ├── 40_Reporting
│   ├── 50_PlatformSystems
│   │
│   └── _archived
│       └── superseded module specs
│
└── guide.md
````

---

## How to Read This Knowledge Base

This knowledge base is intentionally layered.  
Each layer answers a different kind of question.

---

## 1. Domain Models (`domain/`)

**Question answered:**

> **“What must always be true?”**

Domain documents define:
- domain boundaries and ownership
- core concepts and language
- invariants and lifecycle rules
- what the domain owns and what it explicitly does not
- what is in scope vs out of scope

Important rules:
- Domains are **self-contained**
- Domains **do not orchestrate other domains**
- Domains describe **truth**, not timing

Examples:
- Inventory uses a ledger + projection model
- Menu defines composition, not deduction
- Discount defines eligibility, not money calculation

If a rule is not stated in a domain document, it is **not guaranteed** by the system.

---

## 2. Cross-Module Processes (`process/`)

**Question answered:**

> **“How do multiple domains work together?”**

Process documents define:
- business workflows that span domains
- triggers (e.g., Sale Finalized, Sale Voided)
- step ordering and responsibility handoff
- idempotency and retry guarantees
- failure modes and compensating actions

Important rules:
- Processes **own orchestration**, not state
- Processes may be synchronous or event-driven
- Processes are the only place where cross-domain rules live

Examples:
- Finalize Order
- Stock Deduction on Sale Finalization
- Inventory Reversal on Sale Void

If logic spans more than one domain and is **not** in a process doc, it is considered a design smell.

---

## 3. Module Specifications (`modSpec/`)

**Question answered:**

> **“How is this implemented?”**

Module specs translate domains and processes into:
- concrete use cases
- API / command expectations
- read and write responsibilities
- acceptance criteria
- constraints for backend and frontend implementation

Key rules:
- ModSpecs must not invent new business rules
- ModSpecs must align with domain + process documents
- If there is a conflict, **Domain > Process > ModSpec**

ModSpecs are grouped by capability:
- Identity & Access
- Organization & Account
- POS Operations
- Reporting
- Platform Systems (audit, sync, policy)

The `00_guide/version.md` file tracks spec versions and evolution.

---

## Archived Documents (`_archived/`)

The `_archived` folders preserve:
- earlier drafts
- superseded models
- historical thinking

They exist for:
- traceability
- academic reflection
- learning from past decisions

They are **not authoritative**.

---

## Relationship to Playbooks and Templates

This directory contains **knowledge**, not methodology.

- **Playbooks** (in `/playbooks`) define *how we think and model*
- **Templates** (in `/templates`) define *how documents are structured*
- **BusinessLogic** contains the *actual modeled system*

If you want to:
- add a new rule → update a domain
- add a new workflow → write a process
- implement behavior → follow the modspec

---

## Core Architectural Principles (Non-Negotiable)

- Domains do not orchestrate other domains
- Cross-domain behavior must be explicit and documented
- Inventory and cash behavior must be auditable
- Corrections are modeled explicitly (reversal, adjustment), not by mutation
- Business logic favors clarity over cleverness
- Operational reality (human behavior) is acknowledged, not hidden

---

## Scope & Evolution

### Current Scope (Capstone / March Delivery)
- Café-grade POS behavior
- Offline-first, retry-safe workflows
- Inventory that reflects real operations
- Explicit, explainable discount and pricing behavior

### Future Evolution
- Advanced inventory (FEFO, batch enforcement)
- Multi-industry extensions
- Deeper analytics and reporting
- Service extraction (microservices) when justified by scale

New behavior must be introduced by:
1. Updating or adding a **domain model**
2. Defining or extending a **process**
3. Patching the affected **modspecs**

---

## Final Rule

> **If a rule, workflow, or invariant is not documented here, it is not considered part of Modula’s business logic.**

This directory is the **contract** between product intent, system behavior, and implementation.