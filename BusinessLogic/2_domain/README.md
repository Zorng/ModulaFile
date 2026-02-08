# Domain Models — System Truth and Invariants

## Purpose

This directory contains **domain models** for Modula.

Domain models define:
- what concepts exist in the system
- what rules must always be true
- who owns which part of business truth
- how entities change over their lifecycle
- what the system guarantees regardless of implementation details

If a rule is critical to correctness, it **must be stated here**.

---

## What a Domain Model Is

A domain model answers the question:

> **“What must always be true for this part of the system?”**

It describes **system truth**, not behavior over time.

Domain models are:
- stable
- explicit
- independent of technology
- independent of UI and APIs

They exist to prevent ambiguity and hidden assumptions.

---

## What a Domain Model Is NOT

Domain models must **not**:
- orchestrate workflows across domains
- describe timing or sequencing
- describe APIs, endpoints, or UI flows
- describe background jobs or async mechanics
- guess real-world outcomes (e.g., whether an item was physically consumed)

Those belong to:
- `4_process/` (coordination over time)
- `5_modSpec/` (implementation contracts)

If a domain document starts to feel procedural, something is wrong.

---

## Relationship to Other Layers

Domain models sit **between stories and processes**.

The correct conceptual order is:

> **Stories → Domain → Process → ModSpec**

### Stories (`1_stories/`)
- describe human intent and operational reality
- explain *why* the domain exists

### Domain (`2_domain/`)
- defines *what must always be true*
- establishes ownership and boundaries

### Process (`4_process/`)
- defines *how domains coordinate*
- handles timing, triggers, and compensation

### ModSpec (`5_modSpec/`)
- defines *how the system is implemented*
- must obey domain and process rules

**Priority rule:**
Domain > Process > ModSpec

If a ModSpec contradicts a Domain, the ModSpec is wrong.  
If a Process contradicts a Domain, the Process is wrong.

---

## What Each Domain Document Should Contain

A well-formed domain document typically includes:

- **Purpose & Scope**
  - why this domain exists
  - what it owns and what it explicitly does not

- **Core Concepts**
  - entities, value objects, aggregates
  - domain language used consistently elsewhere

- **Invariants**
  - rules that must always hold
  - constraints that cannot be violated

- **Lifecycle**
  - allowed state transitions
  - what actions are irreversible

- **Ownership Boundaries**
  - what other domains may reference
  - what other domains may not modify directly

Not all documents need every section, but **invariants and ownership must be clear**.

---

## Domain Boundaries and Ownership

Each domain:
- owns its own truth
- may expose read-only views to others
- must not mutate another domain’s state directly

Cross-domain behavior must be expressed via:
- explicit processes
- clearly defined events or triggers

Examples:
- Sale owns financial truth
- Inventory owns stock truth
- CashSession owns cash accountability
- Menu owns product definition and intent
- Discount owns eligibility and scope

No domain may silently change another domain’s truth.

---

## Reality vs Intent (Important Principle)

Domains model **system intent**, not physical certainty.

For example:
- Inventory records what the system believes happened
- Humans correct reality via explicit actions (adjustment, force correction)

Domains must **never guess** physical outcomes they cannot observe.

This principle keeps the system honest and auditable.

---

## Organization of This Directory

Domains are grouped by **capability**, not by technical service

## When to Change a Domain Model

You should update a domain model when:
- a new invariant is discovered
- ownership boundaries are unclear
- real-world operations contradict assumptions
- a story exposes a missing rule

You should **not** change a domain model to:
- accommodate implementation shortcuts
- simplify UI flows
- avoid writing a process document

---

## Final Rule

> **If a rule matters to correctness, it belongs in the domain.  
> If it only matters to timing or coordination, it belongs in a process.**

This directory defines the **non-negotiable truth** of Modula.

Code must adapt to domains — not the other way around.
