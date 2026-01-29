# Finalize Order — Cross-Module Process

## 1. Process Overview
Finalize a customer order and produce atomic financial, inventory, and cash effects.

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
- sale_id as idempotency key
- inventory and cash are replay-safe

## 9. Failure Modes
- Retry on transient failure
- Reject on invariant violation

## 10. Postconditions
- Order PAID
- Inventory updated
- Cash recorded
