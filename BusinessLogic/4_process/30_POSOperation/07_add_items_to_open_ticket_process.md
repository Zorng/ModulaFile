# Add Items to Open Ticket (Create New Fulfillment Batch) — POS Operation Process

## Process Name
Add Items to Open Ticket (New Batch)

## Process Type
Cross-Module Business Process (POS Operations)

## Status
Defined (Pay-later baseline; March MVP)

---

## 1. Purpose

In pay-later workflows, customers often add items after the initial order.

This process appends a new **Fulfillment Batch** to an existing Open Ticket without rewriting history.

Each add-items action is a new batch because:
- it prints as a separate kitchen ticket/sticker,
- it has its own immutable snapshots,
- and it avoids per-line fulfillment tracking in March.

---

## 2. Trigger

**Trigger:** Staff selects “Add Items” for an existing Open Ticket.

---

## 3. Preconditions

- Actor is authenticated and authorized for branch operations.
- Branch context is resolved (`tenant_id`, `branch_id`).
- Branch is ACTIVE (not frozen).
- Target Open Ticket exists and is `financial_state = UNPAID`.
- Branch policy `saleAllowPayLater = true`.
- Local draft/cart exists for the add-items action and contains at least one line.
- An OPEN cash session exists for the branch (Capstone I product rule).
- System is online (March baseline: do not silently queue add-items offline).

---

## 4. Orchestration Steps (Happy Path)

### Step 0 — Idempotency Gate

- Apply idempotency gate:
  - `action_key = order.addItems`
  - `idempotency_key = batch_id` (generated once per add-items intent)
- If a batch with this idempotency key already exists:
  - return success (no duplicate batch)

### Step 1 — Resolve Policy Inputs

- Resolve branch policies and effective discounts as required to compute the batch money snapshot:
  - `BusinessLogic/4_process/60_PlatformSystems/20_resolve_branch_policy_process.md`

### Step 2 — Compute and Lock Batch Snapshots

From the add-items draft cart lines, compute and lock:
- order line snapshots
- money snapshot
- policy/discount snapshots needed for later explainability

### Step 3 — Append Batch to Open Ticket

Append `FulfillmentBatch#N` to the existing open ticket:
- `batch_id`
- `placed_at`, `placed_by_user_id`
- `batch_fulfillment_state = IN_PREP`
- locked snapshots from Step 2

### Step 4 — Clear Add-Items Draft

- Clear the add-items draft cart.

### Step 5 — Trigger Kitchen Printing (Best-Effort)

- Trigger kitchen printing for this batch via:
  - `BusinessLogic/4_process/60_PlatformSystems/55_printing_effects_dispatch_process.md`

### Step 6 — Audit

Record audit evidence (minimum):
- `ORDER_ITEMS_ADDED`

---

## 5. Failure / Degradation Rules (March)

- If the ticket is already paid: deny add-items (requires a new sale/order).
- If `saleAllowPayLater = false`: deny new add-items batches (fail closed).
- If offline/unreachable: deny (do not pretend a batch exists).

---

## 6. Related Contracts / Docs

- POS edge case (duplicate add-items):
  - `BusinessLogic/3_contract/10_edgecases/pos_operation_edge_case_sweep_patched.md` (EC-POS-19)

