# Knowing What the System Is Allowed to Do

## Context

During operation, the system must make decisions.

Examples include:
- whether a business can continue operating
- whether certain actions are allowed
- whether limits have been reached

These decisions must be made consistently, even when parts of the system are unavailable.

---

## The Situation

The system needs to know what it is allowed to do **right now**.

This information may depend on:
- the current state of the business relationship
- operational decisions made earlier
- policies defined outside daily workflows

The POS should not need to ask questions about billing or ownership during checkout.

---

## What Operators and Staff Expect

They expect:
- the system to behave consistently
- allowed actions to work without delay
- disallowed actions to be clearly blocked
- the system to continue functioning offline when possible

They do **not** expect:
- operations to depend on real-time external checks
- sudden changes in behavior without explanation
- the system to guess or improvise rules

From their perspective, the system should **already know** what it is allowed to do.

---

## Constraints & Reality

- Connectivity may be unreliable
- Decisions may change over time
- Enforcement must not disrupt daily operations

The system must rely on known, explicit information rather than live assumptions.

---

## How Modula Helps

Modula supports this situation by:
- maintaining a clear snapshot of what is allowed
- using that snapshot consistently across operations
- allowing the snapshot to be updated deliberately
- ensuring that daily workflows do not depend on billing systems

This keeps the POS reliable while allowing commercial rules to evolve.

---

## What This Story Is Not About

- How permissions are calculated
- Where the information comes from
- How often it is refreshed
- Payment or subscription logic

This story is about **having clear, explicit boundaries for system behavior**, so operations remain predictable and trustworthy.