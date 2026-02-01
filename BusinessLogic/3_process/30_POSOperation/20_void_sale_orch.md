# Void Sale — POS Operation Orchestration Process

## Process Name
Void Sale Orchestration

## Process Type
Cross-Module Business Process (POS Operations)

## Status
Defined (Baseline for Capstone 1)

---

## 1. Purpose

This process defines the **authoritative orchestration** for voiding a finalized sale.

Voiding a sale is an **exceptional, policy-driven workflow** that:
- requires explicit authorization
- reverses previously committed business truth
- must remain auditable, idempotent, and safe under retries

This document exists to ensure that void behavior is **explicit and explainable**, not inferred from individual module specs.

---

## 2. Trigger

**Primary trigger:** Manager/Admin approves a void request

> Important:
> - Void *request* does **not** trigger reversals
> - Reversal occurs **only after approval**

This preserves financial correctness and prevents unauthorized truth changes.

---

## 3. Participating Modules & Roles

| Module | Role |
|------|------|
| Sale / Order | Owns sale lifecycle and authorization |
| Inventory | Reverses stock deductions via compensating ledger entries |
| Cash Session | Reverses cash movements (if applicable) |
| Process Layer | Orchestrates sequencing and idempotency |
| Receipt | Reflects void state (read-only) |

---

## 4. Preconditions

- Sale exists
- Sale is in `FINALIZED` or `VOID_PENDING` state
- Void has been **explicitly approved** by an authorized role
- Sale has not already been VOIDED
- Branch context is valid

Failure to meet preconditions aborts the process with **no partial reversals**.

---

## 5. Authoritative Principles

### P-1: Approval is the authorization boundary
- `VOID_PENDING` is a **workflow state**
- `VOIDED` is a **truth change**

No inventory or cash reversal occurs before approval.

---

### P-2: Reversal is compensating, not mutative
- Prior deductions are **not deleted**
- Reversal is performed using **append-only compensating entries**

This preserves auditability and offline safety.

---

### P-3: One idempotency anchor
All reversal effects must be idempotent per:
(branch_id, sale_id)

---

## 6. Orchestration Steps (Happy Path)

### Step 1 — Approve Void (Sale / Order)
- Authorized actor approves void request
- Sale transitions:
  - `VOID_PENDING` → `VOIDED`
- Actor and reason recorded

> This step authorizes truth reversal but does not yet perform effects.

---

### Step 2 — Reverse Inventory Effects
- Trigger inventory reversal sub-process
- Reversal is based on **what was actually deducted**, not menu re-evaluation

See:
- `22_void_sale_inventory_reversal.md`
  (formerly `void_sale_inventory_reversal_process.md`)

---

### Step 3 — Reverse Cash Effects (If Applicable)
- If sale involved cash:
  - append compensating cash movement(s)
- Idempotent per `(branch_id, sale_id)`

(Exact refund mechanics may vary by policy and payment method.)

---

### Step 4 — Complete Void
- Sale is fully VOIDED
- Order/fulfillment record is VOIDED
- Receipt reflects VOIDED state (watermark / status)

At this point, all compensating actions have completed successfully.

---

## 7. Failure Handling

### A. Failure before approval
- Sale remains FINALIZED
- No reversal occurs

---

### B. Failure during reversal
Preferred behavior:
- Sale remains in `VOID_PENDING`
- Reversal steps retry until complete
- System must not partially void

Eventual consistency is acceptable, but **partial truth is not**.

---

## 8. Postconditions

After successful completion:
- Sale state is `VOIDED`
- Inventory ledger contains compensating reversal entries
- Cash ledger reflects reversal (if applicable)
- Receipt and UI reflect voided state
- Audit trail is complete

---

## 9. Related Sub-Processes

- Inventory Reversal:
  - `22_void_sale_inventory_reversal.md`
- Order Void Handling:
  - `21_void_order_process.md`

This orchestration document is the **entry point** for understanding sale void behavior.

---

## 10. Out of Scope

- Partial void (line-level)
- Time-window enforcement (“no void after X hours”)
- Payment gateway refund workflows
- Automation of manager responsiveness

These are **policy and operational concerns**, not orchestration logic.

---

## 11. Design Rationale

Voiding a sale represents a **human-authorized exception**, not a normal flow.

This process:
- respects operational reality
- avoids premature automation
- preserves auditability
- keeps responsibility with people, not toggles

Modula provides the mechanism.
The business decides how (and whether) to use it.

---

# End of Document