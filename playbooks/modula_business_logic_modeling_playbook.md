# Modula Domain Modeling Playbook

*(Canonical – Locked for Capstone II and beyond)*

## **0. Why This Playbook Exists**

This playbook exists to correct a critical design gap discovered during Capstone I:

> We jumped from use cases directly to module specifications and implementation, skipping domain modeling.
> 

This resulted in:

- unclear business logic,
- unstable backend behavior,
- bloated data models,
- excessive patching across modules,
- and technical debt accumulation.

From Capstone II onward, **all Modula development must follow this playbook**.

---

## **1. Correct Design Order (Non-Negotiable)**

All modules must be designed in the following order:

> Use Cases → Domain Modeling → Module Specification → Implementation
> 

Key rule:

- **Module specifications do not define business logic**
- **Domain models do**

If domain logic is unclear, implementation must stop.

---

## **2. Define the Domain Boundary First**

Before modeling anything, clearly define:

- What business problem this module solves
- What it **owns exclusively**
- What it **must not control**

Example:

- Sale domain owns: order lifecycle, pricing, payment intent
- Inventory domain owns: stock state and adjustments
- Cash session domain owns: cash accountability
- Attendance domain owns: time tracking
- Billing domain owns: provisioning and entitlements

> If ownership is ambiguous, the model is not ready.
> 
> If a domain requires outcomes from other domains, it must expose commands or emit events; the orchestration of those interactions belongs in a Cross-Module Process document.

---

## **3. Identify Core Domain Concepts**

List **real-world concepts**, not tables or APIs.

Classify each as:

- **Entity** – has identity and lifecycle
- **Value Object** – immutable, no identity
- **Aggregate Root** – consistency boundary

Example (Order domain):

- Order → Aggregate Root
- OrderItem → Entity
- Payment → Entity
- Money → Value Object
- DiscountSnapshot → Value Object

Rule:

> If you cannot explain the concept to a non-technical business owner, it is not a valid domain concept.
> 

---

## **4. Define Explicit Lifecycle States**

Every entity with behavior must have **explicit states**.

Example (Order):

- DRAFT
- FINALIZED
- VOID_PENDING
- VOIDED
- FULFILLED

For each state, define:

- allowed transitions,
- forbidden transitions,
- who may trigger them,
- side effects (inventory, cash, audit).

Unclear lifecycle = unstable backend.

---

## **5. Model Commands (Intent-Driven)**

Commands represent **intent**, not UI actions.

Examples:

- CreateOrder
- AddItemToOrder
- FinalizeOrder
- RecordPayment
- RequestVoid
- ApproveVoid

Rules:

- Commands validate invariants
- Commands may fail
- Commands produce domain events
- Commands must be idempotent if retried

---

## **6. Define Invariants (Hard Business Rules)**

Invariants are rules that **must never be broken**, even under:

- concurrency,
- offline replay,
- retries,
- crashes.

Examples:

- A finalized order cannot change totals
- Inventory cannot go negative
- A voided sale cannot be paid
- Cash movement requires an open cash session

> Invariants are enforced by the domain,
> 
> 
> **never by UI alone**
> 

---

## **7. Define Domain Events**

Events describe **what has already happened**.

Examples:

- OrderFinalized
- PaymentRecorded
- InventoryDeducted
- CashSessionOpened
- SaleVoided

Events are used for:

- audit logging,
- reporting,
- synchronization,
- future integrations.

---

## **8. Separate Write Model and Read Model**

- **Write model**
    - enforces rules,
    - uses aggregates,
    - strongly consistent.
- **Read model**
    - optimized for UI,
    - may be denormalized,
    - may lag slightly.

Never let UI queries mutate state.

---
## 9. Cross-Module Processes (Application-Orchestrated Logic)

Not all business behavior belongs inside a single domain.

In Modula, **any business workflow that spans multiple domains** (e.g. Order + Inventory + Cash Session) must be modeled as a **Cross-Module Process**, not as part of a domain model.

### Core Rules

- **Domains are self-contained**
  - A domain owns its own state, invariants, lifecycle, commands, and events.
  - A domain must not embed orchestration logic involving other domains.

- **Cross-domain business logic is explicit**
  - Any workflow that coordinates multiple domains must be documented as a **Process**.
  - Processes describe *how domains collaborate*, not *what each domain allows*.

- **Domains expose capabilities, not workflows**
  - Domains expose commands and emit events.
  - Domains do not decide *when* other domains should be invoked.

### Relationship to Domain Modeling

When modeling a domain:

- Define:
  - invariants
  - lifecycle transitions
  - commands accepted
  - events emitted
- Explicitly state:
  - which outcomes are **external effects** (e.g. inventory deduction, cash movement)
  - which responsibilities are **out of scope**

If a behavior requires coordination with other domains, it **does not belong in the domain model** and must be documented as a process instead.

### Examples

**Domain responsibilities (correct):**
- Inventory enforces append-only stock movements and idempotency.
- Order enforces payment and fulfillment state transitions.
- Cash Session enforces one OPEN session per branch.

**Process responsibilities (correct):**
- Finalizing an order (Order + Menu + Inventory + Cash Session)
- Voiding an order (Order + Inventory + Cash Session)
- Offline sync replay across modules

### Documentation Structure

This distinction is reflected in the knowledge base structure:

- **Domain documents**
  - Describe *what must always be true*
  - Located under `BusinessLogic/domain/SelfContained`

- **Process documents**
  - Describe *how multiple domains work together*
  - Located under `BusinessLogic/process/CrossModule`

- **Module specifications**
  - Translate domains and processes into implementable use cases
  - Must not invent cross-domain business logic

## **10. Handle Offline, Concurrency, and Idempotency Explicitly**

For every command, answer:

- Can it be retried?
- Can it be duplicated?
- Can it happen offline?
- What is the idempotency key?
- Who is authoritative (client or server)?

Offline is a **constraint**, not a feature.

---

## **11. Write the Module Specification Last**

A module specification should describe:

- responsibilities,
- guarantees,
- assumptions,
- invariants enforced,
- events emitted,
- failure modes.

A module specification **must not invent business logic**.

---

## **12. Implementation Comes Last**

Only after domain clarity:

- database tables,
- APIs,
- repositories,
- controllers,
- frameworks.

If implementation feels complex:

> the domain model is wrong, not the code.
> 

---

## **13. Guiding Philosophy**

- Modula is not “one big POS”
- It is a set of **business domains that interact**
- Clean boundaries enable scale, refactor, and learning

Restarting the backend to correct domain modeling is **not wasted work** — it is the correct professional decision.

---

### **Status**

- **Locked**
- Applies to all future Modula backend and frontend design
- Serves as foundation for Capstone II