# Place Order (Create Open Ticket) — POS Operation Process

## Process Name
Place Order (Create Open Ticket)

## Process Type
Cross-Module Business Process (POS Operations)

## Status
Defined (Pay-later baseline; March MVP)

---

## 1. Purpose

This process defines how Modula supports **pay-later** workflows (table service) without turning drafts into backend noise.

It creates a persisted, branch-scoped operational record:
- an **Open Ticket** (Order in `UNPAID` state)
- with an initial **Fulfillment Batch** derived from the current local draft

This enables:
- kitchen preparation to start before payment
- staff handover (cashier A places; cashier B settles)
- stable totals that do not change unexpectedly mid-service

---

## 2. Trigger

**Trigger:** Staff selects “Place Order” while using pay-later mode.

---

## 3. Preconditions

- Actor is authenticated and authorized for branch operations.
- Branch context is resolved (`tenant_id`, `branch_id`).
- Branch is ACTIVE (not frozen).
- Branch policy `saleAllowPayLater = true`.
- Local draft/cart exists (device-scoped) and contains at least one line.
- An OPEN cash session exists for the branch (Capstone I product rule).
- System is online (March baseline: do not silently queue place-order offline).

---

## 4. Authoritative Rules (Locked)

- Draft remains client-only. This process persists only the **placed batch snapshot**, not a mutable cart.
- Each placed batch has immutable snapshots:
  - order line snapshot (names/quantities/prices at place time)
  - money snapshot (totals as placed; no later recompute)
- Retrying Place Order must be idempotent.

---

## 5. Orchestration Steps (Happy Path)

### Step 0 — Idempotency Gate

- Apply idempotency gate:
  - `action_key = order.place`
  - `idempotency_key = sale_id` (stable identifier for this order intent)
- If an open ticket already exists for `(branch_id, sale_id)`:
  - return it (no-op)

### Step 1 — Validate Preconditions + Resolve Policy Inputs

- Confirm `saleAllowPayLater = true`.
- Resolve branch policies (VAT/FX/rounding) via:
  - `BusinessLogic/4_process/60_PlatformSystems/20_resolve_branch_policy_process.md`
- Resolve effective discounts (branch-scoped) as required for money snapshot.

### Step 2 — Compute and Lock Batch Snapshots

From the local draft cart lines, compute and lock:
- order line snapshots (item + modifiers + qty + display names)
- money snapshot for this batch (subtotal/discount/VAT/total; tender currency not chosen yet)
- policy/discount snapshots required to explain the money snapshot later

### Step 3 — Persist Open Ticket + Initial Batch

Persist an Order as an Open Ticket:
- `financial_state = UNPAID`
- create `FulfillmentBatch#1` with:
  - `batch_id` (generated at client; stable across retries)
  - `batch_fulfillment_state = IN_PREP`
  - locked snapshots from Step 2

### Step 4 — Clear/Transition Local Draft

- Clear the local draft cart (it has been operationally committed).
- The POS may allow starting a new draft immediately.

### Step 5 — Trigger Kitchen Printing (Best-Effort)

- Trigger printing dispatch (kitchen sticker/ticket) for the new batch:
  - `BusinessLogic/4_process/60_PlatformSystems/55_printing_effects_dispatch_process.md`
- Auto printing must be best-effort and retry-safe (no rollback on print failure).

### Step 6 — Audit

Record audit evidence (minimum):
- `ORDER_PLACED`
- `KITCHEN_PRINT_REQUESTED` (observational; optional)

---

## 6. Failure / Degradation Rules (March)

- If `saleAllowPayLater = false`: deny (fail closed).
- If offline/unreachable: deny and instruct safe alternatives (see EC-POS-20).
- Printing failures never rollback the open ticket creation.

---

## 7. Idempotency Anchor

- Place Order is idempotent per `(branch_id, sale_id)`.
- Batch creation must also be protected from duplicates (batch_id generated once per intent).

---

## 8. Related Contracts / Docs

- Pay-later story:
  - `BusinessLogic/1_stories/selling_and_checkout/placing_order_and_paying_later.md`
- Shift handover story:
  - `BusinessLogic/1_stories/selling_and_checkout/settling_an_unpaid_order_after_shift_change.md`
- POS edge cases:
  - `BusinessLogic/3_contract/10_edgecases/pos_operation_edge_case_sweep_patched.md` (EC-POS-17..20)

