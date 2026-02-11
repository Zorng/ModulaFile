# Cancel Unpaid Ticket (Pay-Later) — POS Operation Process

## Process Name
Cancel Unpaid Ticket

## Process Type
Cross-Module Business Process (POS Operations)

## Status
Defined (March MVP: operational cancellation before payment)

---

## 1. Purpose

In pay-later workflows, not every placed order is eventually paid:
- customer leaves,
- staff placed the wrong order,
- operational mistakes happen.

This process cancels an Open Ticket that has not been settled yet.

Important boundary:
- cancellation is not a refund and not a void (no financial reversal).

---

## 2. Trigger

**Trigger:** Authorized staff selects “Cancel Ticket” on an Open Ticket.

---

## 3. Preconditions

- Actor is authenticated and authorized for branch operations.
- Branch context is resolved (`tenant_id`, `branch_id`).
- Branch is ACTIVE (not frozen).
- Target ticket exists and is `financial_state = UNPAID`.
- A cancellation reason is provided (minimum accountability).

---

## 4. Orchestration Steps (Happy Path)

### Step 0 — Idempotency Gate

- Apply idempotency gate:
  - `action_key = order.cancel`
  - `idempotency_key = sale_id`
- If already cancelled, return success (no-op).

### Step 1 — Mark Ticket Cancelled

- Mark the order/ticket as:
  - `financial_state = CANCELLED` (terminal)
- Mark all batches as `CANCELLED` (if tracked) or record a single cancellation marker.

### Step 2 — Audit

Record audit evidence (minimum):
- `OPEN_TICKET_CANCELLED` (with reason)

---

## 5. Postconditions

- Ticket is cancelled and remains visible for audit/history.
- No cash movements are created.
- No inventory deductions/reversals are created (those are finalize/void concerns).

---

## 6. Related Contracts

- Unpaid cancellation vs paid void:
  - `BusinessLogic/3_contract/10_edgecases/pos_operation_edge_case_sweep_patched.md` (EC-POS-18)

