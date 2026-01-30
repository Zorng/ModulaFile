# Stock Reversal on Sale Void — Cross-Module Process

## Process Name
Void Sale → Inventory Reversal

## Process Type
Cross-Module Business Process

## Status
Defined (Baseline for Capstone 1)

---

## 1. Purpose

This process defines how inventory deductions made during sale finalization are **reversed** when a sale is voided.

It exists to ensure:
- inventory remains **auditable** (no deletion of history)
- reversals are **idempotent** and safe under retries/offline replay
- the system does not rely on “editing” or “removing” prior deductions

This process uses **compensating movements**:
- it does not delete prior SALE_DEDUCTION entries
- it appends VOID_REVERSAL entries that offset the prior deduction

---

## 2. Trigger

**Trigger:** Sale is voided  
(Execution of the *Void Sale* business flow)

Implementation may be synchronous or event-driven (outbox), but rules remain the same.

---

## 3. Participating Modules & Responsibilities

| Module | Responsibility |
|------|----------------|
| Sale / Order | Validates and records that a finalized sale is voided |
| Inventory | Appends reversal movements and updates projections |
| Process Layer | Orchestrates calls and enforces sequencing |
| Menu (Optional) | Not required if reversal is based on recorded deductions; may be used for display only |

---

## 4. Preconditions

- Sale exists
- Sale is in a state eligible for void (typically FINALIZED/PAID)
- Sale has not already been voided (idempotency)
- Branch is active (not frozen), unless business allows void on frozen branches (policy decision)

---

## 5. Authoritative Rule

> **Reversal is based on what was actually deducted for the sale, not on re-evaluating Menu composition.**

Reason:
- Menu items/recipes/modifiers may change after the sale
- re-evaluating composition could produce different results and corrupt inventory truth

Therefore, reversal must reference:
- the Inventory ledger entries created for that sale (SALE_DEDUCTION), or
- a stored “deduction snapshot” captured at finalize time (implementation detail)

Baseline for Capstone 1:
- Inventory can query deduction movements by `(branch_id, sale_id)` and reverse them.

---

## 6. Process Steps (Happy Path)

1. **Void Sale (Sale/Order)**
   - Mark sale VOIDED (or VOID_REQUESTED then VOIDED)
   - Record actor and reason (optional, recommended)

2. **Fetch Deducted Components (Inventory)**
   - Query ledger movements for `(branch_id, sale_id, movement_type = SALE_DEDUCTION)`
   - Aggregate quantities per stock_item_id

3. **Generate Reversal Lines**
   - For each stock_item_id deducted:
     - stock_item_id
     - qty_in_base_unit (same amount as deducted)
     - source = VOID_REVERSAL
     - source_id = sale_id

4. **Apply Inventory Reversal (Inventory)**
   - Append VOID_REVERSAL movements (IN)
   - Enforce idempotency per `(branch_id, sale_id)`
   - Update Branch Stock projection atomically

5. **Complete Void Flow**
   - Cash reversal/refund handling is out of scope here (CashSession/Payment processes)

---

## 7. Idempotency & Retry Rules

This process must be safe under:
- network retries
- offline sync replay
- duplicate void requests

Rules:
- Reversal is idempotent per `(branch_id, sale_id)`
- If reversal already applied:
  - process returns success (no-op)
- Inventory enforces idempotency at the ledger level using:
  - unique constraints on `(source_type, source_id, movement_type)` or
  - a dedicated idempotency_key

---

## 8. Failure Modes & Handling

### A. Sale Void succeeds, Inventory reversal fails
Preferred behavior:
- Void should not be considered complete until reversal succeeds
- Use transactional boundary or outbox with guaranteed processing

If eventual consistency is used:
- Sale may enter VOID_PENDING state until reversal completes

### B. No deduction exists for sale
Possible cases:
- sale contained only NOT_TRACKED components
- sale items had no TRACKED components
Behavior:
- reversal step becomes no-op, still valid

### C. Negative stock concerns
Reversal increases stock, so negative stock is typically not an issue.
If branch is frozen:
- either block reversal (strict), or
- allow reversal (recommended) because it restores stock truth
This is a policy decision; default recommendation is **allow reversal** even if frozen, with audit logging.

---

## 9. Postconditions

After successful completion:
- Sale is VOIDED
- Inventory ledger contains VOID_REVERSAL entries for that sale (if any deduction existed)
- Branch Stock projection reflects the restored stock
- Audit events are recorded

---

## 10. Audit & Observability

Record:
- SALE_VOIDED
- INVENTORY_VOID_REVERSAL_APPLIED

Audit records must include:
- tenant_id
- branch_id
- sale_id
- actor_id
- reason (optional, recommended)

---

## 11. Out of Scope

- Partial void (void some lines only)
- Refund/payment gateway reversal rules
- Complex time-window constraints (“cannot void after X hours”)
- Inventory adjustments unrelated to void

---

## 12. Design Notes (Why This Exists)

- Finalize Sale deducts inventory from TRACKED components.
- Void Sale must reverse **what actually happened**, not what would happen now.
- Ledger-based inventory requires compensating movements, not edits.

This keeps inventory correct even when:
- recipes change after the sale
- modifiers/catalog evolve
- offline sync replays requests
