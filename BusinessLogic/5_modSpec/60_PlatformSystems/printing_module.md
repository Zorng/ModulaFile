# Printing Module — Platform Module (Pilot: Receipt + Kitchen Sticker)

**Version:** 1.0  
**Status:** Draft (March pilot scope: receipt printer + kitchen sticker printer)  
**Module Type:** Platform / Supporting Module (Operational Effects)  
**Depends on:** Tenant & Branch Context, Access Control, Receipt (snapshot), Order/Fulfillment (snapshot), Offline Sync (local cache), Audit Logging (observational)  
**Related:** Sale finalize orchestration (triggers auto print), Operational Notification (optional awareness)

---

## 1. Purpose

The Printing module provides a consistent way for Modula to perform **hardware printing** without letting printing leak into business truth.

It supports two pilot outputs:
- customer receipt printing (front counter)
- kitchen sticker printing (back of house)

Printing is **best-effort**:
- printing failures must never invalidate finalized sales,
- printing must be retryable and re-printable,
- and printing uses immutable snapshots (receipt/order) so it does not recompute truth.

Authoritative domain:
- `BusinessLogic/2_domain/60_PlatformSystems/printing_and_peripherals_domain.md`

Authoritative process:
- `BusinessLogic/4_process/60_PlatformSystems/55_printing_effects_dispatch_process.md`

Related POS entry point:
- `BusinessLogic/4_process/30_POSOperation/10_finalize_sale_orch.md` (Step 8 operational effects)

---

## 2. Scope (March Pilot)

Included:
- routing print intents by **printer role**:
  - `RECEIPT_PRINTER`
  - `KITCHEN_STICKER_PRINTER`
- auto printing after finalize sale (best-effort, retry-safe)
- manual print/reprint from receipt/order views
- printer-not-configured degradation (no blocking of operations)

Excluded (explicit):
- terminal identity model (register/device identity)
- cash drawer control
- advanced printer configuration UI
- remote print management across multiple devices

---

## 3. Core Concepts

### 3.1 Printer Role (Pilot)
- `RECEIPT_PRINTER`
- `KITCHEN_STICKER_PRINTER`

### 3.2 Print Kind
- `RECEIPT` (payload from receipt snapshot)
- `KITCHEN_STICKER` (payload from order snapshot)

### 3.3 Printer Assignment (Pilot-Level)

For the pilot, printer assignment is allowed to be **device-local configuration**:
- the POS device stores which physical printer is used for each role.

Rationale:
- avoids requiring terminal identity or cross-device printer registries for March.

---

## 4. Interfaces (Conceptual)

- `PrintReceipt(tenant_id, branch_id, actor_id, sale_id | receipt_id, purpose)`
- `PrintKitchenSticker(tenant_id, branch_id, actor_id, sale_id | order_id, purpose, batch_id?)`

Where `purpose` is:
- `AUTO_AFTER_FINALIZE`
- `MANUAL_REPRINT`

---

## 5. Authorization (Read-Safe Printing)

Printing must be gated by Access Control, but should remain usable for history even when billing is frozen.

Guidance:
- `receipt.print` should be treated as a **READ-effect** action (it does not mutate business truth).
- Kitchen printing should use an explicit action key (recommended): `order.print.kitchen` and should also be treated as **READ-effect**.

This allows:
- printing/reprinting receipts for historical evidence while `FROZEN`,
- without opening operational write surfaces.

---

## 6. Idempotency & Retry Safety

### 6.1 Auto printing (best-effort idempotency)

Auto-after-finalize print attempts should be deduped best-effort using `(branch_id, sale_id, print_kind)` as an anchor.

Recommended idempotency keys:
- `print:receipt:{branch_id}:{sale_id}`
- `print:kitchen:{branch_id}:{sale_id}` (pay-first)
- `print:kitchen:{branch_id}:{sale_id}:{batch_id}` (pay-later batch prints)

### 6.2 Manual reprint

Manual reprints are explicit operator intent and should **not** be suppressed by idempotency.

---

## 7. Offline Behavior (Pilot)

Printing may be needed during unstable connectivity.

Rules:
- If the receipt/order snapshot is available locally (cached/derived), printing may proceed.
- Printing while offline must not be treated as backend-confirmed evidence.
- If the underlying finalize operation later fails to sync, the system must surface that discrepancy clearly (operational follow-up).

---

## 8. Audit & Observability (Observational)

Printing events are observational and must not drive business truth.

Baseline event types:
- Receipt:
  - `RECEIPT_PRINT_REQUESTED`
- Kitchen:
  - `KITCHEN_STICKER_PRINT_REQUESTED`

Optional (recommended for support):
- `PRINT_FAILED` with metadata `{ print_kind, reason_code }`

---

## 9. Out of Scope (Future)

- printer fleet management and tenant-wide printer profiles
- per-item routing (different kitchen stations)
- print templates designer
- label printer formatting engines
