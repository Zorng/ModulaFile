# 55 — Printing Effects Dispatch (Receipt + Kitchen Sticker)

## Purpose

This process defines how Modula performs printing as **best-effort operational effects**:
- receipt printing (front counter)
- kitchen sticker printing (back of house)

It exists to keep one rule true:

> Printing must never be a prerequisite for business truth.

Finalize-sale commits truth first, then triggers printing. Printing may fail and be retried/reprinted safely.

---

## Story Traceability

Derived primarily from:
- `BusinessLogic/1_stories/selling_and_checkout/completeing_a_sale.md`
- `BusinessLogic/1_stories/selling_and_checkout/prep_and_deliver_order.md`

---

## Domains Involved

- Printing & Peripherals (effect vocabulary + invariants)
- Receipt (receipt snapshot payload)
- Order/Fulfillment (order snapshot payload)
- Audit (observational evidence)
- Offline Sync + Idempotency (retry safety)

References:
- Domain: `BusinessLogic/2_domain/60_PlatformSystems/printing_and_peripherals_domain.md`
- Edge cases: `BusinessLogic/3_contract/10_edgecases/printing_effects_edge_case_sweep.md`

---

## When This Process Runs

Triggered:
- automatically after a sale finalizes successfully (auto printing), and/or
- manually by a user choosing "Print" or "Reprint".

---

## Inputs (Minimum, Conceptual)

- `tenant_id`
- `branch_id`
- `actor_id` (or `SYSTEM` for auto)
- `print_kind` (`RECEIPT` | `KITCHEN_STICKER`)
- `purpose` (`AUTO_AFTER_FINALIZE` | `MANUAL_REPRINT`)
- `source_anchor`:
  - `sale_id` (stable idempotency anchor)
  - and either `receipt_id` (for receipt) or `order_id` (for kitchen)

---

## Canonical Rules (Locked)

### R-PRN-1 — Printing is observational
- Printing must not mutate sale/order/cash/inventory truth.
- Failures must not invalidate finalized sales.

### R-PRN-2 — Payloads come from snapshots
- Receipt prints use the immutable receipt snapshot.
- Kitchen prints use the immutable order snapshot.

### R-PRN-3 — Best-effort idempotency for auto printing

Auto printing must avoid spamming duplicates under retries.

Recommended idempotency keys (conceptual):
- `print:receipt:{branch_id}:{sale_id}`
- `print:kitchen:{branch_id}:{sale_id}`

Manual reprints bypass this suppression by design (explicit operator intent).

---

## Orchestration Steps

### Step 1 — Validate context and authorization (if interactive)

- If this is a user-initiated print:
  - validate actor is authenticated
  - authorize:
    - `receipt.print` for receipt printing, or
    - `order.print.kitchen` (recommended action key) for kitchen printing
- If this is auto-after-finalize:
  - treat it as `SYSTEM` initiated and skip user authorization (finalize already authorized).

---

### Step 2 — Resolve printer assignment (pilot-level)

Resolve where the print should go:
- `RECEIPT` -> `RECEIPT_PRINTER`
- `KITCHEN_STICKER` -> `KITCHEN_STICKER_PRINTER`

If the mapping is missing:
- return a "printer not configured" operational failure
- do not block operations (sale is already finalized)

---

### Step 3 — Resolve print payload from snapshots

- For `RECEIPT`:
  - load receipt snapshot by `receipt_id` (or by `(branch_id, sale_id)` if needed)
- For `KITCHEN_STICKER`:
  - load order snapshot by `order_id` (or by `(branch_id, sale_id)` if needed)

If snapshot cannot be loaded:
- return failure with `DEPENDENCY_MISSING`
- allow retry once data is available.

---

### Step 4 — Apply idempotency suppression (auto printing only)

If `purpose = AUTO_AFTER_FINALIZE`:
- apply best-effort idempotency using the key in R-PRN-3
- if already printed, return success (no-op)

If `purpose = MANUAL_REPRINT`:
- do not suppress; always attempt printing.

---

### Step 5 — Dispatch to printing adapter

Send the payload to the printing adapter (implementation concern):
- adapter returns `SUCCESS` or `FAILED`
- failures include a stable `reason_code` (printer offline, connection lost, etc.)

---

### Step 6 — Observability (audit / signals)

Record observational audit events (best-effort):
- `RECEIPT_PRINT_REQUESTED` (receipt)
- `KITCHEN_STICKER_PRINT_REQUESTED` (kitchen)

On failure, record:
- `PRINT_FAILED` with metadata `{ print_kind, reason_code }` (optional for March; recommended for support)

These events must not be used to drive business truth.

---

## Failure / Degradation Rules (March)

- Never rollback a finalized sale due to printing issues.
- Always allow manual reprint from receipt/order detail views.
- Kitchen staff must have an on-screen Orders fallback; printing is a convenience.

