# BusinessLogic Knowledge Base

## Purpose

This directory is the **authoritative knowledge base for Modula’s business behavior**.

It defines:
- what the system is meant to support in the real world
- what rules must always be true
- how different parts of the system cooperate
- what developers (human or AI) are allowed to implement

If a behavior exists in code, it **must be traceable** to a document here.

This directory is intentionally written to be:
- understandable by developers
- grounded in real operational reality
- defensible in academic review
- stable enough to guide long-term evolution

---

## What This Directory Is (and Is Not)

### This directory **IS**:
- the single source of truth for business rules and workflows
- a shared mental model for the team
- a bridge between product thinking and implementation
- a contract that code must obey

### This directory **IS NOT**:
- infrastructure or DevOps documentation
- deployment guides
- coding style or framework instructions
- sprint tickets or task breakdowns

Those belong elsewhere.

---

## High-Level Structure
```
BusinessLogic
├── README.MD
│
├── 1_stories
│   ├── README.md
│   ├── correcting_mistakes
│   ├── handling_branding
│   ├── handling_menu
│   ├── handling_money
│   ├── handling_staff&attendance
│   ├── managing_stock
│   ├── running_promotions
│   ├── saas_governance
│   └── selling_and_checkout
│
├── 2_domain
│   ├── README.md
│   ├── 10_Identity&Authorization
│   ├── 20_OrgAccount
│   │   ├── tenant_domain_consistency_patched.md
│   │   └── branch_domain.md
│   ├── 30_HR
│   ├── 40_POSOperation
│   ├── 50_Reporting
│   ├── 60_PlatformSystems
│   └── _archived
│
├── 3_contract
│   ├── 10_edgecases
│   ├── 20_ux_specs
│   └── _archive
│
├── 4_process
│   ├── README.md
│   ├── 10_WorkForce
│   ├── 20_IdentityAccess
│   ├── 20_OrgAccount
│   ├── 30_POSOperation
│   ├── 50_Reporting
│   └── 60_PlatformSystems
│
├── 5_modSpec
│   ├── 00_guide
│   ├── 10_IdentityAccess
│   ├── 20_OrgAccount
│   ├── 30_HR
│   ├── 40_POSOperation
│   ├── 50_Reporting
│   ├── 60_PlatformSystems
│   └── _archived
│
└── _maps
    └── *_story_coverage_map.md
```

---

## How to Read This Knowledge Base (Very Important)

This knowledge base is **layered by intent**.  
Each layer answers a different kind of question.

The correct reading and design order is:

> **Stories → Domain → Contract → Process → ModSpec**

Never skip a layer.

---

## 1. Stories (`1_stories/`) — Human Reality

**Question answered:**

> **“What is actually happening in the real world?”**

Stories describe:
- human intent and expectations
- operational context
- normal work, mistakes, and exceptions
- what people reasonably expect the system to support
- what the system explicitly does *not* try to solve

Key rules:
- stories are non-technical
- stories must not explain system mechanics
- stories may cross domains and modules freely
- stories are grouped by **human context**, not architecture

Examples:
- completing a sale at the counter
- voiding a sale when a mistake happens
- closing the cash drawer
- adjusting stock when reality differs

Stories represent **user requirements**, not system design.

---

## 2. Domain Models (`2_domain/`) — System Truth

**Question answered:**

> **“What must always be true?”**

Domain documents define:
- ownership and boundaries
- core concepts and language
- invariants and lifecycle rules
- what the domain owns and what it explicitly does not
- why the model exists (context matters)

Key rules:
- domains are self-contained
- domains do not orchestrate other domains
- domains describe **truth**, not timing
- domains do not describe APIs or workflows

Examples:
- Inventory uses a ledger + projection model
- Menu defines composition and tracking intent
- Discount defines eligibility and scope, not calculation

If a rule is not stated in a domain document, it is **not guaranteed** by the system.

---

## 3. Contracts (`3_contract/`) — Cross-Layer Agreements

**Question answered:**

> **“What edge cases and cross-layer behaviors must be handled consistently?”**

Contract documents define:
- cross-layer agreements (Backend/Frontend/QA share the same expectations)
- edge cases and boundary safety rules (explicit March vs later stance)
- required behavioral states (errors, degraded behavior, audit signals)

Key rules:
- contracts do not replace domains (truth/invariants) or processes (sequencing/orchestration)
- contracts must not prescribe UI layouts; they define behavior the UI must support
- conflicts are resolved as: **Domain > Contract > Process > ModSpec**

---

## 4. Cross-Module Processes (`4_process/`) — Coordination Over Time

**Question answered:**

> **“How do multiple domains work together?”**

Process documents define:
- workflows that span more than one domain
- triggers (e.g. sale finalized, sale voided)
- step ordering and responsibility handoff
- idempotency and retry guarantees
- failure modes and compensating actions

Key rules:
- processes own orchestration, not state
- processes may be synchronous or event-driven
- cross-domain rules must live here

Examples:
- finalize order
- stock deduction on sale finalization
- inventory reversal on sale void

If logic spans more than one domain and is **not** in a process doc, it is a design smell.

---

## 5. Module Specifications (`5_modSpec/`) — Implementation Contract

**Question answered:**

> **“How is this implemented?”**

ModSpecs translate stories, domains, and processes into:
- concrete system use cases (UC-*)
- commands and responsibilities
- backend and frontend expectations
- acceptance criteria
- constraints for implementation

Key rules:
- modspecs must not invent new business rules
- modspecs must align with domain, contract, and process docs
- conflicts are resolved as: **Domain > Contract > Process > ModSpec**

ModSpecs are grouped by capability:
- identity & access
- organization & account
- POS operations
- reporting
- platform systems (audit, sync, policy)

The `00_guide/version.md` file tracks spec evolution.

Note:
- `5_modSpec/60_PlatformSystems/` contains legacy artifacts. Policy + OperationalNotification have been aligned to domain/contract/process; Audit + Offline Sync remain legacy until their PlatformSystems domain models exist.

---

## Archived Documents (`_archived/`)

Archived folders preserve:
- earlier drafts
- superseded thinking
- historical decisions

They exist for:
- traceability
- academic reflection
- learning from past mistakes

They are **not authoritative**.

Never delete history; archive it.

---

## Relationship to Playbooks and Templates

- **Playbooks** explain *how to think* (modeling philosophy)
- **Templates** explain *how to write*
- **BusinessLogic** defines *what the system is*

When adding or changing behavior:
1. clarify the real-world problem (story)
2. patch or add a domain model
3. add or update a process if cross-domain
4. patch the affected modspecs
5. archive superseded documents

Never reverse this order.

---

## Core Architectural Principles (Non-Negotiable)

- domains do not orchestrate other domains
- cross-domain behavior must be explicit
- inventory and cash behavior must be auditable
- corrections are explicit (reversal, adjustment), not silent mutation
- business logic favors clarity over cleverness
- the system records intent; humans correct reality

---

## Scope & Evolution

### Current Scope (Capstone / March Delivery)
- café-grade POS behavior
- offline-first, retry-safe workflows
- inventory that reflects real operations
- explicit, explainable discount behavior

### Future Evolution
- advanced inventory (FEFO, stricter batch rules)
- multi-industry extensions
- deeper analytics and reporting
- service extraction when justified by scale

New behavior must always be introduced through this knowledge base.

---

## Final Rule

> **If a rule, workflow, or invariant is not documented here, it is not considered part of Modula’s business logic.**

This directory is the **contract** between product intent, system behavior, and implementation.
