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

Related contracts:
- `BusinessLogic/3_contract/10_edgecases/pos_operation_edge_case_sweep_patched.md`
- `BusinessLogic/3_contract/10_edgecases/identity_hr_pos_boundary_edge_cases_patched.md` (HR ↔ POS boundary)
- KHQR payment confirmation (pre-finalize):
  - `BusinessLogic/4_process/30_POSOperation/05_khqr_payment_confirmation_process.md`

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
| Payment (KHQR verification) | Confirms external payment proof when method is KHQR |
| Receipt | Persists immutable receipt snapshot |
| Hardware / Effects | Printing, drawer opening (best-effort) |
| Process Layer | Enforces sequencing and idempotency |

---

## 4. Preconditions

- Sale exists and is in a **finalizable** state
- Sale has not already been finalized (idempotency)
- Branch is active (not frozen)
- An OPEN cash session exists for the branch (Capstone 1 product rule; required even for non-cash sales)
- Required payment method information is present
- If payment method is KHQR:
  - a valid KHQR tracking key (`md5`) is provided
  - payment proof can be confirmed by the backend and matches expected `amount + currency + receiver`
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

`(branch_id, sale_id)`

Notes:
- `sale_id` is a stable identifier generated once per draft cart and reused for:
  - network retries (double-submit)
  - offline finalize replay during sync
  - outbox/event retries (if used)
- Every cross-module write MUST be replay-safe using the same anchor:
  - cash movement append
  - inventory ledger append
  - receipt snapshot creation
  - order creation
- The platform idempotency gate is defined in:
  - `BusinessLogic/4_process/60_PlatformSystems/80_idempotency_gate_process.md`

---

### P-4: Attendance is not a hard prerequisite for financial truth (March)
Finalize Sale must not be blocked solely because attendance is missing.

If the actor has no ACTIVE attendance session:
- allow finalize (if authorized)
- emit an audit signal (example: `SALE_FINALIZED_WITHOUT_ATTENDANCE`)

This matches the boundary contract in `BusinessLogic/3_contract/10_edgecases/identity_hr_pos_boundary_edge_cases_patched.md`.

---

## 6. Orchestration Steps (Happy Path)

### Step 0 — Idempotency Gate (Process Layer)
- Apply the platform idempotency gate using:
  - `action_key = sale.finalize`
  - `idempotency_key = sale_id` (stable per draft cart)
- If a finalized sale already exists for `(branch_id, sale_id)`:
  - return the existing finalized snapshot (no-op)
- Otherwise:
  - begin the orchestration

This prevents duplicates under:
- double-tap
- network retries
- offline replay

---

### Step 1 — Validate Preconditions + Resolve Context
Validate:
- actor is authenticated and authorized to finalize sales in this tenant/branch
- branch is ACTIVE (not frozen)
- an OPEN cash session exists for the branch (Capstone 1 product rule)
- cart/draft contains at least one line item
- payment method and tender details are valid
- if payment method is KHQR:
  - verify payment status via backend confirmation using `md5`
  - validate proof binds to expected `amount + currency + receiver`

Load required read facts:
- branch policy snapshot inputs (VAT/FX/Rounding)
- applicable discount rules (branch-scoped)
- menu metadata required for line evaluation (including modifier selections)

---

### Step 2 — Compute and Lock Sale Truth (Sale/Order)
Sale module computes and locks:
- line totals
- discount snapshot (which rules applied + their parameters)
- policy snapshot (VAT/FX/Rounding values used)
- totals (grand totals + currency rounding)
- payment snapshot (method + tender + change, if applicable)

Then persist the sale as:
- `status = FINALIZED`
- immutable snapshots attached

This step is the financial point-of-truth.

---

### Step 3 — Create Fulfillment Order (Sale/Order)
Create an Order linked to the finalized sale:
- initial status: `IN_PREP`
- order lines copied from the finalized sale snapshot

The Order is operational/fulfillment-only; it must not alter sale totals.

---

### Step 4 — Record Cash Movement (CashSession, Conditional)
If payment includes cash:
- append `SALE_IN` cash movement linked to `(branch_id, sale_id)`
- enforce idempotency per the same anchor

If payment is non-cash (e.g., KHQR):
- no cash ledger mutation occurs

---

### Step 5 — Execute Inventory Deduction (Inventory, Conditional)
If sale evaluation produces `deduction_lines` (tracked components from menu mapping):
- run the inventory deduction sub-process:
  - `BusinessLogic/4_process/30_POSOperation/13_stock_deduction_on_finalize_sale_process.md`

Key rule reminder:
- deduction occurs only when evaluated composition contains TRACKED components
- idempotent per `(branch_id, sale_id)`

---

### Step 6 — Create Receipt Snapshot (Receipt)
- derive receipt snapshot from the finalized sale snapshot
- persist receipt idempotently per `(branch_id, sale_id)`

Receipt is an immutable view of truth; it does not decide truth.

---

### Step 7 — Emit Audit + Domain Signals
At minimum, write audit records for:
- `SALE_FINALIZED`
- `CASH_MOVEMENT_RECORDED` (if cash)
- `INVENTORY_DEDUCTION_APPLIED` (if applicable)
- `RECEIPT_CREATED`

If attendance is missing, also record:
- `SALE_FINALIZED_WITHOUT_ATTENDANCE` (audit signal)

---

### Step 8 — Trigger Operational Effects (Best-Effort)
After truth is committed:
- request receipt printing
- request kitchen ticket printing (if used)
- request drawer opening (if cash)

Failures here:
- must be retryable
- must not invalidate the finalized sale

---

## 7. Transaction Boundary (March-Preferred)

### Option A — Single Transaction (Modular Monolith)
Recommended for March:
- Steps 1–7 execute inside one transactional boundary.
- If any truth mutation fails (cash/inventory/receipt/order), the sale must not finalize.

### Option B — Outbox / Event-Driven
Allowed if used consistently:
- Step 2 commits sale truth + an outbox record atomically.
- Downstream steps consume outbox events idempotently using the same anchor.
- Until all truth mutations complete, the system must not expose “partially finalized” sales as complete.

---

## 8. Failure Modes & Handling

### A. Duplicate finalize request (retry/double-submit)
- Detect by idempotency anchor → return existing finalized snapshot (no-op).

### B. Preconditions fail (frozen branch, missing cash session, invalid payment)
- Reject finalize.
- No sale/order/ledger/receipt is written.

### C. Cash movement write fails (cash payments)
- Finalize must fail/rollback (truth must not be partial).
- If event-driven, remain in a non-complete state until cash movement is recorded.

### D. Inventory deduction fails (when enabled)
- Finalize must fail/rollback.
- No partial inventory movements are allowed.

### E. Receipt creation fails
- Finalize must fail/rollback (receipt is part of immutable truth package).

### F. Hardware failures (print/drawer)
- Do not rollback finalize.
- Record an operational error signal and allow retry/reprint.

---

## 9. Postconditions
After successful completion:
- Sale is `FINALIZED` and immutable.
- Order is created in `IN_PREP` and references the sale.
- Cash movement is recorded exactly once (if cash).
- Inventory deduction is recorded exactly once (if enabled and applicable).
- Receipt snapshot exists exactly once.
- Audit trail is complete.

---

## 10. Related Sub-Processes
- Finalize Order (legacy summary):
  - `BusinessLogic/4_process/30_POSOperation/12_finalizeOrder_process.md`
- Inventory deduction on finalize:
  - `BusinessLogic/4_process/30_POSOperation/13_stock_deduction_on_finalize_sale_process.md`
- Device draft constraints:
  - `BusinessLogic/4_process/30_POSOperation/11_deviceDraft_process.md`
- Void/reversal entry point:
  - `BusinessLogic/4_process/30_POSOperation/20_void_sale_orch.md`

This orchestration document is the **entry point** for understanding finalize-sale behavior.

---

## 11. Out of Scope (Capstone 1)
- Split payments
- Partial refunds
- Payment gateway settlement
- Partial finalize (line-level)
- Complex promotions requiring “best-of” selection

---

## 12. Design Notes
- Finalize Sale is where immutable truth is locked.
- Inventory and cash are append-only ledgers; idempotency is mandatory.
- Operational hardware effects are not allowed to corrupt financial truth.

---

# End of Document
