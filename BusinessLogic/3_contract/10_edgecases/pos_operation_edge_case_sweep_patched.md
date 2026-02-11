# Edge Case Contract — POS Operations (Sale, Cash, Inventory, Effects)

## Metadata
- **Contract Type**: Edge Cases
- **Scope**: POS operations within a tenant+branch context (finalize sale, void sale, cash accountability, inventory effects, hardware effects)
- **Primary Audience**: Backend, Frontend, QA
- **Owner(s)**: Finalize Sale Orchestration, Void Sale Orchestration, Order, Cash Session, Inventory, Menu, Discount, Audit, Hardware/Effects
- **Last Updated**: 2026-02-06
- **Delivery Level**:
  - **March**: must be handled or explicitly degraded safely
  - **Later**: explicitly deferred
- **Related Docs**:
  - `BusinessLogic/4_process/30_POSOperation/05_khqr_payment_confirmation_process.md`
  - `BusinessLogic/4_process/30_POSOperation/10_finalize_sale_orch.md`
  - `BusinessLogic/4_process/30_POSOperation/13_stock_deduction_on_finalize_sale_process.md`
  - `BusinessLogic/4_process/30_POSOperation/20_void_sale_orch.md`
  - `BusinessLogic/4_process/30_POSOperation/22_void_sale_inventory_reversal_process.md`
  - `BusinessLogic/4_process/30_POSOperation/23_void_sale_cash_reversal_process.md`
  - `BusinessLogic/3_contract/10_edgecases/identity_hr_pos_boundary_edge_cases_patched.md` (HR ↔ POS boundary)

---

## Purpose

This document captures **POS operation edge cases** that require consistent behavior across:
- Order/Sale truth
- Cash accountability (Cash Session)
- Inventory effects (deduction + reversal)
- Receipt/hardware effects (best-effort)

It exists to prevent “hidden logic” scattered across module specs and to make March behavior testable by QA.

---

## Scope

### In-scope
- finalize sale retry/idempotency behavior
- finalize sale failure boundaries (what must rollback vs what is best-effort)
- inventory deduction/reversal edge cases
- void orchestration edge cases (approval boundary + compensation)
- hardware/effects degradation rules (print/drawer failures)
- device draft safety rules when exiting operational context

### Out-of-scope (for this contract)
- payment gateway refunds / settlements
- returns after end-of-day close (separate “refund” workflow, future)
- multi-register drawer models
- advanced reporting pipelines

---

## Definitions / Legend

- **Owner**: where the decision belongs (domain/process)
- **March**: must be handled or explicitly degraded safely
- **Signal**: audit/log/UX signal expected
- **Idempotency anchor**: `(branch_id, sale_id)` for finalize + void cascades
- **Open Ticket**: a persisted order in `UNPAID` state created by “place order” (pay-later workflow)
- **Fulfillment Batch**: a batch of items added at place-order or add-items; kitchen printing is batch-scoped

---

## Edge Case Catalog

### EC-POS-01 — Duplicate Finalize Sale (Double-Tap / Retry / Offline Replay)
- **Scenario**: Cashier finalizes the same sale twice due to double-tap, network retry, or offline sync replay.
- **Trigger**: Duplicate finalize request for the same `(branch_id, sale_id)`.
- **Expected Behavior**:
  - Return the already-finalized snapshot (no-op).
  - No duplicate cash movements, inventory movements, receipts, or orders are created.
- **Owner**: Finalize Sale orchestration + participating modules via idempotency
- **March**: Yes

### EC-POS-02 — Hardware Effects Fail After Truth Is Committed
- **Scenario**: Receipt printing fails, kitchen printer fails, or cash drawer open fails after finalize succeeds.
- **Trigger**: Operational effects step fails (printer offline, paper jam, drawer not connected).
- **Expected Behavior**:
  - Sale remains finalized (truth must not be rolled back due to hardware).
  - Effects are best-effort and retryable.
- **Owner**: Finalize Sale orchestration + Hardware/Effects
- **March**: Yes

### EC-POS-03 — Finalize Sale Without an Open Cash Session
- **Scenario**: Cashier tries to finalize a sale when there is no OPEN cash session for the branch.
- **Trigger**: Finalize sale precondition check fails.
- **Expected Behavior**:
  - Reject finalize sale (no partial truth is written).
  - UI message should instruct to open a cash session first.
- **Owner**: Finalize Sale orchestration + Cash Session
- **March**: Yes

### EC-POS-04 — Inventory Deduction Fails During Finalize Sale
- **Scenario**: Sale truth is ready to finalize, but inventory deduction fails (DB error, invariant violation).
- **Trigger**: Inventory deduction step fails.
- **Expected Behavior**:
  - Finalize sale must not complete “partially”.
  - If March uses a single transaction boundary: rollback and reject finalize.
  - If March uses outbox/eventual processing: sale must not be shown as fully finalized until deduction completes (use an explicit “pending finalize” state).
- **Owner**: Finalize Sale orchestration + Inventory
- **March**: Yes

### EC-POS-05 — Sale Has No TRACKED Components (No Inventory Effects)
- **Scenario**: Sale contains only NOT_TRACKED components (or items without tracked composition).
- **Trigger**: Inventory deduction step evaluates to “nothing to deduct”.
- **Expected Behavior**:
  - Inventory deduction becomes a no-op.
  - Finalize sale proceeds normally.
- **Owner**: Inventory deduction process + Inventory
- **March**: Yes

### EC-POS-06 — Negative Stock Result After Deduction
- **Scenario**: A deduction would make on-hand stock negative (because of drift, late restock entry, or operational reality).
- **Trigger**: Inventory projection after deduction is below zero.
- **Expected Behavior (baseline)**:
  - Allow finalize sale, record the deduction, and emit a warning/audit signal.
  - (Future) tenants may enable strict prevention as a policy.
- **Owner**: Inventory domain (policy) + Finalize Sale orchestration (enforcement point)
- **March**: Yes (baseline allow-with-signal)
- **Signal**: `NEGATIVE_STOCK_AFTER_SALE_DEDUCTION` (or equivalent audit entry)

### EC-POS-07 — Void Approved Twice / Duplicate Void Orchestration
- **Scenario**: Manager approves void twice, or void orchestration is retried.
- **Trigger**: Duplicate void approval or retry for the same `(branch_id, sale_id)`.
- **Expected Behavior**:
  - Idempotent: return “already voided” result.
  - No duplicate inventory reversal or cash reversal movements are created.
- **Owner**: Void Sale orchestration + Inventory + Cash Session
- **March**: Yes

### EC-POS-08 — Menu/Recipe Changed After Finalize (Void Must Still Be Correct)
- **Scenario**: Menu composition is edited after a sale was finalized, then the sale is voided later.
- **Trigger**: Void sale triggers inventory reversal.
- **Expected Behavior**:
  - Inventory reversal must be based on what was actually deducted (ledger/deduction snapshot), not on re-evaluating current menu composition.
- **Owner**: Void Sale → Inventory Reversal process + Inventory
- **March**: Yes

### EC-POS-09 — Void Approved but Inventory Reversal Fails
- **Scenario**: Sale is approved for void, but inventory reversal step fails.
- **Trigger**: Inventory reversal step fails.
- **Expected Behavior**:
  - Avoid “partial void” exposure.
  - Prefer `VOID_PENDING` until reversal completes (or a single transactional boundary).
  - System retries reversal idempotently until completion; audit the failure.
- **Owner**: Void Sale orchestration + Inventory
- **March**: Yes

### EC-POS-10 — Device Draft Exists When Exiting Operational Context
- **Scenario**: A device has an in-progress draft cart/order when the user tries to switch branch, log out, or close the cash session.
- **Trigger**: Context-exit action occurs while a draft exists.
- **Expected Behavior**:
  - Draft must be resolved (finalize, discard, or save) before exit.
  - Prevents silent loss of intent and reduces double-submit edge cases.
- **Owner**: Application/UX layer + Device draft process
- **March**: Yes

### EC-POS-11 — Void Cash Sale When Related Cash Session Is CLOSED
- **Scenario**: Manager/Admin attempts to approve a void for a cash sale that belongs to a CLOSED cash session.
- **Trigger**: Approve void request while `sale.cash_session_id` is not OPEN.
- **Expected Behavior**:
  - Deny void approval for March baseline.
  - Surface a clear UX message: “This sale belongs to a closed cash session.”
  - Direct the business to a separate refund workflow (explicitly deferred).
- **Owner**: Void Sale orchestration + Cash Session
- **March**: Yes

### EC-POS-12 — Void Approved but Cash Refund Recording Fails
- **Scenario**: Void is approved/executed, but the cash refund movement cannot be recorded (DB error, invariant violation).
- **Trigger**: Cash refund step fails.
- **Expected Behavior**:
  - Avoid “partial void” exposure.
  - Prefer `VOID_PENDING` until refund is recorded successfully (or a single transactional boundary).
  - System retries idempotently; audit the failure.
- **Owner**: Void Sale orchestration + Cash Session
- **March**: Yes

### EC-POS-13 — KHQR Paid but Sale Not Finalized (Connectivity Loss / App Reload)
- **Scenario**: Customer completes KHQR payment successfully, but the cashier device loses connection or reloads before finalizing the sale.
- **Trigger**: Payment is confirmed by the network, but `Finalize Sale` was not executed.
- **Expected Behavior (March baseline)**:
  - The POS must not block the cashier from starting a new sale just because KHQR confirmation cannot be performed immediately.
  - The cashier must be able to **park** the in-flight KHQR payment as `PENDING_CONFIRMATION` (not finalized) and continue operating.
  - The system must allow recovery by re-checking payment status using the KHQR tracking key (`md5`) when connectivity returns.
  - Sale finalization remains manual and safe to retry; no duplicate finalized sales are created for the same sale intent.
- **Owner**: KHQR payment confirmation process + Finalize Sale orchestration
- **March**: Yes

### EC-POS-14 — KHQR Payment Proof Mismatch (Wrong Amount/Currency/Receiver)
- **Scenario**: A KHQR payment is reported as paid, but the proof does not match the sale being finalized (wrong amount, wrong currency, or wrong receiver account).
- **Trigger**: Backend verification returns a paid transaction whose `amount/currency/toAccountId` does not match expected values.
- **Expected Behavior**:
  - For March, KHQR amount must not be cashier-entered; QR generation must use the computed payable amount to reduce operator error.
  - Reject finalize; no partial truth is written.
  - Emit an audit/operational signal for investigation (future: manager resolution workflow).
- **Owner**: Payment domain validation + Finalize Sale orchestration
- **March**: Yes (reject + signal)

### EC-POS-15 — KHQR Unpaid / Expired
- **Scenario**: Cashier generates a KHQR code but the customer does not complete payment before the payment window closes.
- **Trigger**: Confirmation checks never reach "paid" and the QR expires.
- **Expected Behavior**:
  - Sale is not finalizable.
  - Allow the cashier to regenerate a new KHQR attempt for the same sale intent.
  - Only the latest attempt is eligible for finalize; older attempts become `SUPERSEDED`.
- **Owner**: KHQR payment confirmation process
- **March**: Yes

### EC-POS-16 — Superseded KHQR Attempt Is Paid Later
- **Scenario**: The cashier regenerates KHQR (creating a new attempt), but a customer later pays an older (superseded) QR attempt.
- **Trigger**: Backend confirmation detects "paid" for a `SUPERSEDED` attempt `md5`.
- **Expected Behavior (March baseline)**:
  - Do not finalize any sale automatically from a superseded attempt.
  - Emit an operational notification + audit signal for investigation and business handling (refund/settlement workflows are deferred).
- **Owner**: KHQR payment confirmation process + Audit + OperationalNotification
- **March**: Yes (signal-only; no automated resolution)

### EC-POS-17 — Close Cash Session With Unpaid Tickets
- **Scenario**: Staff attempts to close the cash session while there are open tickets (orders in `UNPAID` state).
- **Trigger**: Close cash session action.
- **Expected Behavior (March baseline)**:
  - Deny close with a clear message: “There are unpaid orders. Settle or cancel them before closing.”
  - Provide a way to list the unpaid orders so staff can resolve them.
- **Owner**: Cash Session + POSOperation processes
- **March**: Yes

### EC-POS-18 — Cancel Unpaid Ticket vs Void Paid Sale (Different Workflows)
- **Scenario**: An order was placed but the customer leaves before paying, or the staff needs to terminate it before settlement.
- **Trigger**: Staff tries to “void” an order that is still `UNPAID`.
- **Expected Behavior (March baseline)**:
  - If `UNPAID`: allow cancellation (no cash/inventory reversal; record reason; audit it).
  - If `PAID`: use the existing void workflow (approval boundary + compensating effects).
- **Owner**: Order/Sale domain + POSOperation processes + Audit
- **March**: Yes

### EC-POS-19 — Duplicate Add-Items Must Not Create Duplicate Batches/Prints
- **Scenario**: Cashier double-taps “Add Items” or retries due to unstable connectivity.
- **Trigger**: Duplicate add-items request for the same batch intent.
- **Expected Behavior (March baseline)**:
  - Only one fulfillment batch is created.
  - Kitchen auto-print for that batch is best-effort idempotent (dedupe by `(branch_id, sale_id, batch_id)` or equivalent).
  - Manual reprint remains allowed and explicit.
- **Owner**: Order (batch creation) + Printing effects (dedupe) + Idempotency gate
- **March**: Yes

### EC-POS-20 — Pay-Later Enabled but Network Unavailable
- **Scenario**: The branch enables pay-later, but connectivity is unstable and “place order” cannot reach the backend.
- **Trigger**: Attempt to place an order while offline/unreachable.
- **Expected Behavior (March baseline)**:
  - Deny pay-later placement (do not pretend an open ticket was created).
  - Provide a safe degradation path:
    - either use pay-first offline flows, or
    - wait until connectivity returns.
- **Owner**: Offline Sync + POSOperation processes
- **March**: Yes (explicit degradation)

---

## March-Locked Policy (Refund vs Void)

- For March baseline, **void approval is blocked** when the related cash session is CLOSED.
- “Refund after close” is treated as a **separate workflow** (explicitly deferred; see contract out-of-scope).

---

## Open Questions (Intentionally Not Locked Here Yet)

- **Branch frozen semantics for corrective writes**:
  - Finalize sale should be blocked when branch is frozen.
  - Corrective reversals (inventory reversal, void completion) may need to be allow-listed with audit (policy decision).

---

## Summary

This contract provides a shared list of POS-critical edge cases with expected behavior for March.
Processes (`BusinessLogic/4_process/30_POSOperation/*.md`) should reference these cases so QA can test behavior without reverse-engineering module specs.

_End of POS Operations edge case contract_
