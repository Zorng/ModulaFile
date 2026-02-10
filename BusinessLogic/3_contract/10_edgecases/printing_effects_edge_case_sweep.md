# Printing Effects — Edge Case Sweep (Receipt + Kitchen Sticker)

## Purpose

This contract locks printing behavior so printing does not drift into:
- blocking finalize-sale,
- creating duplicate/confusing kitchen tickets,
- or becoming a hidden prerequisite for business truth.

Printing is an **operational effect** and must remain best-effort.

Authoritative domain:
- `BusinessLogic/2_domain/60_PlatformSystems/printing_and_peripherals_domain.md`

Related POS contract (print failures after truth commit):
- `BusinessLogic/3_contract/10_edgecases/pos_operation_edge_case_sweep_patched.md` (EC-POS-02)

---

## Edge Cases (Locked)

### PRN-EC-01 — Receipt/Kitchen print fails after sale is finalized

**Scenario:** Sale finalizes successfully, but printing fails (printer offline, out of paper).

**Required behavior:**
- Sale remains finalized; no rollback.
- UI shows a clear "print failed" signal and offers retry/reprint.
- Failure should be observable (audit/telemetry signal), but must not block operations.

---

### PRN-EC-02 — Duplicate finalize retry must not spam auto prints

**Scenario:** Cashier double-taps finalize or the client retries finalize due to network uncertainty.

**Required behavior:**
- Finalize sale remains idempotent (already locked in POS process).
- Auto-after-finalize print intents should be deduped best-effort (by `(branch_id, sale_id, print_kind)`).
- Manual reprint remains allowed and is not suppressed.

---

### PRN-EC-03 — Kitchen sticker printed twice (risk of double preparation)

**Scenario:** Kitchen sticker prints twice due to retries or unclear success.

**Required behavior (March baseline):**
- Kitchen staff must have a non-print fallback source of truth (Orders list).
- Duplicate kitchen print must not create a second order.
- If the system cannot guarantee exactly-once physical print, it must clearly label:
  - order number and timestamp on the sticker so duplicates are recognizable.

---

### PRN-EC-04 — Printer mapping missing or misconfigured

**Scenario:** Receipt printer role is not configured on the device, or kitchen printer is missing.

**Required behavior:**
- Do not block sale finalization.
- Offer alternative: on-screen receipt view and manual print later.
- Store enough identifiers to allow reprint once configured (sale_id/receipt_id/order_id).

---

### PRN-EC-05 — Offline finalize + printing

**Scenario:** Cash sale is finalized offline (queued for sync), and printing is requested.

**Required behavior:**
- Printing may proceed using the locally available sale/receipt/order snapshot.
- Offline printing must not be treated as backend-confirmed evidence.
- When sync later rejects the finalize operation, the system must surface that discrepancy clearly.

---

### PRN-EC-06 — Reprint after void

**Scenario:** A voided sale's receipt is reprinted.

**Required behavior:**
- Receipt remains immutable; print must include the VOIDED / VOID_PENDING visual state.
- Reprinting must not trigger any financial/inventory changes.

