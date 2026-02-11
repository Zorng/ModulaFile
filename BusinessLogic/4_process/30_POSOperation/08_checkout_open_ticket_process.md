# Checkout Open Ticket (Settle Pay-Later) — POS Operation Process

## Process Name
Checkout Open Ticket (Settle Pay-Later)

## Process Type
Cross-Module Business Process (POS Operations)

## Status
Defined (March MVP: cash + KHQR; manual finalize)

---

## 1. Purpose

This process settles an existing Open Ticket (an order in `UNPAID` state) into an immutable finalized sale record.

Key design goal:
- allow Cashier B to settle an order placed by Cashier A,
- without rewriting placed batches,
- while preserving the existing finalize-sale integrity rules (cash/inventory/receipt idempotency).

---

## 2. Trigger

**Trigger:** Staff selects “Checkout” on an Open Ticket.

---

## 3. Preconditions

- Actor is authenticated and authorized to finalize sales in the branch.
- Branch context is resolved (`tenant_id`, `branch_id`).
- Branch is ACTIVE (not frozen).
- Open Ticket exists and is `financial_state = UNPAID`.
- An OPEN cash session exists for the branch (Capstone I product rule).
- Payment method selection is valid:
  - CASH: tender currency + amount received (for change)
  - KHQR: tender currency; payment confirmation required before finalize

Note:
- This checkout is allowed even if `saleAllowPayLater` was later toggled off (safe closure of existing tickets).

---

## 4. Authoritative Rules (Locked)

- Checkout produces the same immutable truth package as pay-first finalize:
  - sale snapshot
  - receipt snapshot
  - cash movement (if cash)
  - inventory deduction (if enabled/applicable)
- Totals must be explainable and stable:
  - payable amount is computed from the ticket’s stored batch money snapshots (no surprise recompute).
- Checkout is idempotent per `(branch_id, sale_id)`.

---

## 5. Orchestration Steps (Happy Path)

### Step 0 — Resolve Ticket Total

- Load the Open Ticket.
- Compute the payable amount from stored snapshots:
  - sum batch money snapshots into a ticket total view

### Step 1 — Payment Confirmation (If KHQR)

- Follow the KHQR confirmation process using the ticket payable total:
  - `BusinessLogic/4_process/30_POSOperation/05_khqr_payment_confirmation_process.md`
- Do not finalize until payment is confirmed.

### Step 2 — Finalize Sale (Entry Point)

Call the canonical finalize-sale orchestration with this ticket’s `sale_id`:
- `BusinessLogic/4_process/30_POSOperation/10_finalize_sale_orch.md`

Finalize-sale must treat this as “checkout from existing ticket”:
- sale snapshot is created for the existing `sale_id`
- the existing order/ticket is marked `PAID` and linked to the sale snapshot
- receipt/cash/inventory behavior remains unchanged

### Step 3 — Post-Checkout State

After finalize succeeds:
- Open Ticket financial state becomes `PAID` (terminal for March; void is separate)
- ticket remains visible in Orders history (read-only)

### Step 4 — Audit

Record audit evidence (minimum):
- `SALE_FINALIZED`
- additional ticket settlement evidence (recommended): `OPEN_TICKET_SETTLED`

---

## 6. Failure / Degradation Rules (March)

- If ticket is already paid: treat as idempotent no-op (return existing finalized sale snapshot).
- If cash session is not OPEN: deny checkout (same as pay-first).
- If KHQR cannot be confirmed due to network: allow parking pending confirmation (KHQR contract), but do not finalize.

---

## 7. Related Docs

- Finalize-sale entry point:
  - `BusinessLogic/4_process/30_POSOperation/10_finalize_sale_orch.md`
- POS edge case (cash session close with unpaid tickets):
  - `BusinessLogic/3_contract/10_edgecases/pos_operation_edge_case_sweep_patched.md` (EC-POS-17)

