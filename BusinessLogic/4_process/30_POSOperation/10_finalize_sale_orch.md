# Finalize Sale — POS Operation Orchestration Process

## Process Name
Finalize Sale Orchestration

## Process Type
Cross-Module Business Process (POS Operations)

## Status
Defined (Baseline for Capstone 1)

---

## 1. Purpose

This process defines the **authoritative orchestration** that occurs when a sale is finalized in Modula.

It is the **spine of POS operations**, responsible for:
- locking monetary truth
- coordinating cross-module effects
- separating business truth from operational side-effects
- ensuring idempotency, auditability, and offline safety

This document exists because finalize-sale behavior spans multiple domains and **must not be inferred by reading individual module specs**.

---

## 2. Trigger

**Trigger:** Cashier initiates *Finalize Sale* action  
(e.g. “Pay”, “Complete Sale”)

This process may be implemented:
- synchronously (single transaction), or
- asynchronously (event/outbox driven)

The trigger mechanism does **not** change the business rules below.

---

## 3. Participating Modules & Roles

| Module | Role |
|------|------|
| Sale / Order | Owns sale lifecycle and monetary truth |
| Discount | Determines eligible discount rules |
| Menu | Provides composition metadata |
| Inventory | Records stock movements (ledger + projection) |
| CashSession | Records cash movements |
| Receipt | Persists immutable receipt snapshot |
| Hardware / Effects | Printing, drawer opening (best-effort) |
| Process Layer | Enforces sequencing and idempotency |

---

## 4. Preconditions

- Sale exists and is in a **finalizable** state
- Sale has not already been finalized (idempotency)
- Branch is active (not frozen)
- Required payment method information is present
- Required domain data (Menu, Discount, Inventory) is accessible

Failure to meet any precondition aborts the process with **no partial truth written**.

---

## 5. Authoritative Principles (Non-Negotiable)

### P-1: Finalize Sale is the point of truth
After finalization:
- prices
- discounts
- totals
- inventory effects
- cash effects

must never change retroactively.

---

### P-2: Separate truth from effects

**Truth changes (must succeed or rollback):**
- Sale snapshot creation
- Discount application snapshot
- Cash movement recording
- Inventory deduction recording
- Receipt snapshot creation

**Operational effects (best-effort, retryable):**
- receipt printing
- kitchen ticket / sticker printing
- cash drawer opening

Hardware failures must **not** invalidate finalized sales.

---

### P-3: One idempotency anchor
All downstream operations must be idempotent using: