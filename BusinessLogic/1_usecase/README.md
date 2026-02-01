# Use Cases & Stories — Human-Centered Requirements

## Purpose

This directory contains **use cases and stories** that describe Modula from a **human and operational perspective**.

These documents capture:
- real situations that happen in a café or store
- the intent, tension, and expectations of people
- operational reality that cannot be derived from code or system structure

This layer represents the outcome of **user requirement gathering and requirement engineering**.

It exists to explain **why** the system behaves the way it does — before any technical modeling begins.

---

## What This Layer Is (and Is Not)

### This layer **IS**:
- human-centered, non-technical narratives
- written in plain language
- grounded in real operational context
- a guide for domain modeling and process design
- allowed to cross system boundaries freely

### This layer **IS NOT**:
- a domain model
- a process specification
- an API or system design
- an implementation guide
- organized by modules or services

If you see entities, states, APIs, or schemas here, something has gone wrong.

---

## Core Principle (Non-Negotiable)

> **Use cases must not be organized by system modules.**

Why:
- modules are a *system decomposition*
- stories are a *human experience decomposition*

Humans do not think in terms of:
- Sale module
- Inventory module
- CashSession module

They think in terms of:
- serving a customer
- correcting a mistake
- handling money
- starting or ending a shift
- dealing with something unusual

Stories must follow **human context**, not architecture.

---

## How Stories Are Organized

Stories are grouped by **operational context**, not by feature or module.

Examples of valid groupings:
- selling and checkout
- handling money
- correcting mistakes
- managing staff presence
- closing the day
- dealing with exceptions

Each folder represents a **coherent human situation space**.

A story may involve:
- one module
- many modules
- multiple categories of modules

That is expected and correct.

---

## Story Granularity

### One story = one human situation

A story should describe:
- one primary situation
- from the perspective of people involved
- even if multiple system actions occur

Stories are **not**:
- one per module
- one per technical use case
- one per API endpoint

Some stories naturally span multiple technical use cases.  
Some modules naturally participate in multiple stories.

This is normal.

---

## Suggested Story Structure (Guideline, Not Template)

Stories do not follow a strict template, but they usually include:

1. **Context**
   - who is involved
   - what is happening operationally

2. **The Problem or Tension**
   - what goes wrong or requires attention
   - why this situation is not trivial

3. **Desired Outcome**
   - what the human expects to happen
   - from their point of view

4. **Constraints & Reality**
   - human limitations
   - policy, timing, physical constraints
   - what the system cannot magically solve

5. **How Modula Helps**
   - what the system supports
   - what responsibility remains with people

6. **Non-Goals / Misconceptions**
   - what this story is not about
   - common misunderstandings to avoid

Not all sections are required.  
Narrative clarity is more important than completeness.

---

## Relationship to Other Layers

The correct reading and design order is:

> **Use Case / Story → Domain → Process → ModSpec**

- Stories explain **human intent**
- Domains define **system truth**
- Processes define **cross-module behavior**
- ModSpecs define **implementation**

Stories should inform domain modeling, not be derived from it.

---

## Traceability (Lightweight)

Stories may optionally include a short section such as:

> **System Areas Touched (Informational)**  
> Sale, Inventory, CashSession, Receipt, OperationalNotification

This is for reader orientation only.  
It must not be used to organize or constrain stories.

---

## Evolution & Archival

Stories may evolve as:
- operations change
- product understanding improves
- new insights emerge

When a story becomes obsolete:
- move it to `_archived/`
- do not delete it
- preserve learning and rationale

---

## Final Rule

> **If a story reads like system documentation, it does not belong here.**

This directory exists to keep Modula grounded in **real human work**, not just correct software.