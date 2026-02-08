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

Related contracts:
- `BusinessLogic/3_contract/10_edgecases/pos_operation_edge_case_sweep_patched.md`
- `BusinessLogic/3_contract/10_edgecases/identity_hr_pos_boundary_edge_cases_patched.md` (HR ↔ POS boundary)
- Operational notifications (best-effort signals):
  - `BusinessLogic/4_process/60_PlatformSystems/30_operational_notification_emission_process.md`

---

## 2. Trigger

**Primary trigger:** Manager/Admin approves a void request

> Important:
> - Void *request* does **not** trigger reversals
> - Reversal occurs **only after approval**

This preserves financial correctness and prevents unauthorized truth changes.

OperationalNotification note:
- When a void is requested and the Sale transitions to `VOID_PENDING`, the system should emit a best-effort in-app notification to the approver pool (see OperationalNotification process). This is a signal only; approval must still re-check current state.

---

## 3. Participating Modules & Roles

| Module | Role |
|------|------|
| Sale / Order | Owns sale lifecycle and authorization |
| Inventory | Reverses stock deductions via compensating ledger entries |
| Cash Session | Reverses cash movements (if applicable) |
| Process Layer | Orchestrates sequencing and idempotency |
| Receipt | Reflects void state (read-only) |
| OperationalNotification | Emits best-effort in-app signals to approvers/requester |

---

## 4. Preconditions

- Sale exists
- Sale is in `FINALIZED` or `VOID_PENDING` state
- Void has been **explicitly approved** by an authorized role
- Sale has not already been VOIDED
- Branch context is valid
- March baseline business rules:
  - Payment method must be CASH (QR void/refund is blocked)
  - Related cash session must be OPEN (void is blocked if session is CLOSED)

Failure to meet preconditions aborts the process with **no partial reversals**.

---

## 5. Authoritative Principles

### P-1: Approval is the authorization boundary
- `VOID_PENDING` is a **workflow state**
- `VOIDED` is a **truth change**

No inventory or cash reversal occurs before approval.

**March safety rule:** sale becomes `VOIDED` only when required compensating effects (inventory + cash) are recorded successfully.

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

### Step 1 — Approve Void + Lock Execution (Sale / Order)
- Authorized actor approves void request
- Validate March business rules (CASH only + related cash session OPEN)
- Actor and reason recorded

> This step authorizes truth reversal but does not yet perform effects.

---

### Step 2 — Reverse Inventory Effects
- Trigger inventory reversal sub-process
- Reversal is based on **what was actually deducted**, not menu re-evaluation

See:
- `22_void_sale_inventory_reversal_process.md`

---

### Step 3 — Reverse Cash Effects (If Applicable)
- Trigger cash refund sub-process
- Idempotent per `(branch_id, sale_id)`

Note: In our ledger model, “cash reversal” is implemented by recording a compensating cash-out movement (e.g., `REFUND_CASH`) that offsets the original sale cash-in. There is no separate “extra refund” beyond this movement.

See:
- `23_void_sale_cash_reversal_process.md`

---

### Step 4 — Complete Void
- Persist void state changes:
  - Sale status → `VOIDED`
  - Order status → `VOIDED`
  - Void request status → APPROVED
- Sale is fully VOIDED
- Order/fulfillment record is VOIDED
- Receipt reflects VOIDED state (watermark / status)

At this point, all compensating actions have completed successfully.

---

### Step 5 — Emit Operational Notification (Best-Effort)
- Notify the requester that the void was approved (ON-02).
- Emission must be idempotent per `(branch_id, sale_id)` and must not be a correctness dependency for the void itself.

See:
- `BusinessLogic/4_process/60_PlatformSystems/30_operational_notification_emission_process.md`

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
  - `22_void_sale_inventory_reversal_process.md`
- Cash Refund:
  - `23_void_sale_cash_reversal_process.md`
- Order Void Handling:
  - `21_voidOrder_process.md`

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
