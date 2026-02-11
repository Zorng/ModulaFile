# Finalize Order — Cross-Module Process

## 1. Process Overview
Finalize a customer order and produce atomic financial, inventory, and cash effects.

> Note:
> - This file is a legacy summary view kept for reference.
> - The canonical entry point is `BusinessLogic/4_process/30_POSOperation/10_finalize_sale_orch.md`.
> - With pay-later enabled, an Order/Open Ticket may already exist in `UNPAID` state before checkout.

## 2. Trigger and Actors
- Trigger: Cashier finalizes order
- Actor: Cashier

## 3. Participating Domains
- Order: validates and transitions order
- Menu: computes deduction intent
- Inventory: applies stock ledger entries
- Cash Session: records cash movement

## 4. Orchestrator
FinalizeOrderService (application layer)

## 5. Preconditions
- Order exists in draft-equivalent state
- Cash session is OPEN
- Branch is active (not frozen)
- Required snapshots available

## 6. Process Steps
1. Validate order invariants
2. Mark order as PAID
3. Compute inventory deltas
4. Apply inventory deductions (idempotent)
5. Record cash movement
6. Emit ORDER_FINALIZED

## 7. Transaction Boundary
Single transactional boundary (modular monolith)

## 8. Idempotency
- (branch_id, sale_id) as idempotency key
- inventory and cash are replay-safe

## 9. Failure Modes
- Retry on transient failure
- Reject on invariant violation

## 10. Postconditions
- Order PAID
- Inventory updated
- Cash recorded

## Related
- Entry point orchestration:
  - `10_finalize_sale_orch.md`
- Inventory deduction detail:
  - `13_stock_deduction_on_finalize_sale_process.md`
