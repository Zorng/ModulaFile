# Cross-Module Process Template

## 1. Process Overview

**Process Name:**  
**Business Goal:**  

Describe *what real-world business outcome this process achieves*.

This document defines a **cross-module business process** that coordinates multiple self-contained domains.  
It does **not** own persistent state and does **not** redefine domain invariants.

---

## 2. Trigger and Actors

### Trigger
Describe what initiates this process:
- User action (e.g. cashier finalizes an order)
- System event (e.g. offline sync replay)
- Scheduled operation (e.g. end-of-day close)

### Actors
- Primary actor(s)
- Secondary/system actors (if any)

---

## 3. Participating Domains

List all domains involved and their responsibilities **within this process**.

| Domain | Responsibility in this Process |
|------|--------------------------------|
| Order | Owns order state transitions |
| Inventory | Applies stock movements idempotently |
| Cash Session | Records cash movements |
| Menu | Provides recipe / deduction computation |
| (Others) | |

**Important:**  
Each domain enforces its own invariants.  
This process only **orchestrates** calls between them.

---

## 4. Orchestrator

**Orchestrator Name:**  
(e.g. `FinalizeOrderService`, `VoidOrderWorkflow`)

Describe:
- Where this orchestrator lives (application layer)
- Whether it is synchronous or event-driven
- Whether it runs inside a single transaction or coordinates multiple steps

---

## 5. Preconditions

Conditions that must be true **before** the process starts.

Examples:
- Order exists and is in DRAFT state
- Branch is active
- Cash session is OPEN
- Required identifiers/snapshots are available

Preconditions are **validated**, not created, by this process.

---

## 6. Process Steps (Happy Path)

Describe the **ordered steps** of the process.

Example format:

1. Validate order state and invariants (Order domain)
2. Record payment / mark order as PAID (Order domain)
3. Compute inventory deductions (Menu domain)
4. Apply inventory movements (Inventory domain)
5. Record cash movement (Cash Session domain)
6. Emit completion event

Each step should:
- Call a domain command
- Rely on that domain to enforce its own rules

---

## 7. Transaction Boundary

Describe how atomicity is handled.

- Single database transaction (modular monolith)
- Multiple transactions with compensation
- Outbox/event-driven pattern

State explicitly:
- What must be atomic
- What can be retried safely

---

## 8. Idempotency and Retry Strategy

Define how the process behaves under retries and duplicate execution.

Include:
- Idempotency keys used
- Which steps are idempotent by domain guarantee
- How out-of-order or repeated execution is handled

Example:
- Inventory deductions are idempotent per `(source_type, source_id)`
- Cash movements reject duplicates
- Order finalization is idempotent

---

## 9. Failure Modes and Compensation

Describe what happens when steps fail.

Examples:
- Inventory deduction fails
- Cash movement fails
- Process crashes mid-execution
- Offline replay causes duplicate execution

For each failure:
- Is the process retried?
- Is compensation required?
- Is manual intervention needed?

---

## 10. Postconditions

Conditions that must be true **after** successful completion.

Examples:
- Order is in PAID state
- Inventory reflects deductions
- Cash session balance updated
- Audit events recorded

---

## 11. Emitted Events and Audit Trail

List any events emitted by the process or domains as a result.

Examples:
- ORDER_FINALIZED
- INVENTORY_DEDUCTED
- CASH_MOVEMENT_RECORDED

Clarify:
- Which events are domain events
- Which are audit/logging only

---

## 12. Out of Scope

Explicitly state what this process does **not** handle.

Examples:
- Refund settlement with external payment gateway
- Advanced inventory allocation (FEFO/FIFO)
- Reporting or analytics

This prevents scope creep and ambiguity.

---

## 13. Future Extensions

Optional section describing:
- Likely future enhancements
- What would need to change in domains or orchestration
- Any known limitations of the current process

---

## Notes

- This document describes **how domains collaborate**, not how they are implemented.
- Domain invariants must never be redefined here.
- If a rule belongs to one domain only, it should live in that domain’s model instead.