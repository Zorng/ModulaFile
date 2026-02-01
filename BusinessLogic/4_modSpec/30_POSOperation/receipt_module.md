# Receipt / eReceipt + Basic Customization — Feature Module (Capstone I)

**Version:** 1.0  
**Status:** Defined (Aligned with Capstone 1 boundaries)  
**Module Type:** Feature Module  

**Depends on:**  
Tenant & Branch Context (Core)  
Sale (Feature)  
Auth & Authorization (Core)  
Policy & Configuration (Core)  
Sync & Offline Support (Core)  
Audit Logging (Core)

---

## 1. Purpose

Provide a customer-facing **eReceipt** (and print-ready receipt layout) for every **finalized sale**, so staff can quickly view, print, and resolve disputes. Support minimal receipt customization via a **per-branch footer**, while all brand/address/contact identity is sourced from **Tenant & Branch Context**.

Receipt presentation must be consistent with Sale and Order state (e.g., void pending, voided).

---

## 2. Scope & Boundaries

### Includes
- View eReceipt for a finalized sale via **Orders list** (primary entry point)
- Print-ready receipt layout (browser print for Capstone 1)
- Receipt “snapshot” behavior: receipts remain consistent over time
- Branch-level receipt footer customization (per branch)
- Display of taxes, discounts, rounding results, and dual-currency totals as computed by Sale
- Handling of void-related states in receipt display (VOID_PENDING / VOIDED)
- Offline viewing/sync-safe receipt access (no duplicates)

### Excludes
- Managing business logo, brand name, branch address/contact (owned by Tenant & Branch module)
- Payment settlement records and external payment verification
- Advanced printing hardware integrations (thermal printer control)
- Digital delivery (email/SMS) of receipts

---

## 3. Key Decisions (Locked)

- **Identity Source of Truth:** Brand, logo, branch name, address, contact are pulled from **Tenant & Branch Context**.
- **Receipt Snapshot:** On sale finalization, Receipt stores a **snapshot** of identity fields + footer used for that sale to avoid historical receipts changing.
- **Customization:** Only receipt-specific customization in Capstone 1 is **footer text per branch**.

---

## 4. Use Cases

### UC-1 — View eReceipt from Orders
**Actors:** Cashier, Manager, Admin

**Preconditions**
- User is authenticated
- Branch context is active
- At least one finalized sale exists (order exists)

**Main Flow**
1. User opens Orders list in their portal.
2. User selects an order.
3. System opens the eReceipt view for the sale linked to the order.
4. eReceipt displays:
   - branch identity (from snapshot)
   - sale details and totals (from sale snapshot)
   - order status (fulfillment) and sale status (financial)

**Postconditions**
- None (read-only)

---

### UC-2 — Print Receipt (Web Print)
**Actors:** Cashier, Manager, Admin

**Preconditions**
- eReceipt is visible for a finalized sale

**Main Flow**
1. User selects “Print”.
2. System renders print-friendly layout.
3. Browser print dialog is opened.

**Postconditions**
- None (printing is an external action)

**Notes**
- Printing should be allowed for FINALIZED receipts.
- Printing behavior for VOIDED receipts is implementation-owned, but receipt must show VOIDED watermark clearly.

---

### UC-3 — Customize Receipt Footer (Per Branch)
**Actors:** Admin

**Preconditions**
- Admin authenticated
- Tenant context active
- Branch exists

**Main Flow**
1. Admin opens Receipt Settings (or Receipt Customization).
2. Admin selects a branch.
3. Admin edits footer text (e.g., “Thank you”, return policy).
4. Admin previews the receipt footer.
5. Admin saves changes.
6. System stores the footer configuration for that branch.

**Postconditions**
- Future receipts for that branch use the updated footer (snapshot stored at finalize).

---

### UC-4 — Render Receipt Snapshot on Finalize
**Actors:** System

**Preconditions**
- Sale is being finalized successfully (Sale module checkout complete)

**Main Flow**
1. System collects receipt identity fields from Tenant & Branch context:
   - tenant/brand display name
   - logo URL (if any)
   - branch name
   - branch address
   - branch contact
2. System reads receipt footer configuration for the branch.
3. System stores a **receipt snapshot** linked to the finalized sale.
4. Receipt snapshot becomes the source of truth for viewing/printing that sale’s receipt.

**Postconditions**
- Receipt identity and footer are frozen for that sale.
- Future edits to branch profile or footer do not change historical receipts.

---

### UC-5 — Receipt Display for VOID_PENDING / VOIDED
**Actors:** Cashier, Manager, Admin

**Preconditions**
- Sale exists and has status VOID_PENDING or VOIDED

**Main Flow**
1. User opens the order receipt.
2. If sale is VOID_PENDING:
   - receipt displays “Void Requested” indicator (non-destructive)
3. If sale is VOIDED:
   - receipt displays “VOIDED” watermark
   - receipt displays void reason (if available) and approval info (if available)
4. Receipt totals remain as originally finalized (immutability); void is shown as state, not recomputation.

**Postconditions**
- None (read-only)

---

## 5. Requirements

- R1: eReceipt must be available for every finalized sale.
- R2: eReceipt must be accessible from Orders (tap order → receipt).
- R3: Receipt must display dual-currency totals and rounding results exactly as stored in the finalized sale record.
- R4: Receipt must display sale type (dine-in / takeaway / delivery).
- R5: Receipt must display item lines including modifiers/add-ons and item-level discounts as recorded by Sale.
- R6: Receipt must display branch-wide discount and VAT effects as recorded by Sale.
- R7: Receipt must use Tenant & Branch identity as source of truth at finalize time and store a snapshot.
- R8: Admin can configure a **per-branch footer** and preview it.
- R9: Historical receipts must remain consistent after branch profile or footer updates.
- R10: Receipt must reflect sale status:
  - FINALIZED: normal receipt
  - VOID_PENDING: “Void requested” indicator
  - VOIDED: “VOIDED” watermark + void details (where applicable)
- R11: Receipt view/print must be read-only and must not mutate sale or order state.

---

## 6. Acceptance Criteria

- AC-1: Selecting an order opens its eReceipt.
- AC-2: Receipt shows branch identity (name/address/contact/logo) correctly from the stored snapshot.
- AC-3: Receipt shows all totals and discounts exactly matching the finalized sale.
- AC-4: Admin can set footer text per branch and preview it.
- AC-5: Updating footer affects future receipts only (historical receipts unchanged).
- AC-6: VOID_PENDING receipts show a “Void requested” indicator.
- AC-7: VOIDED receipts show a clear “VOIDED” watermark and void metadata where available.
- AC-8: Receipt is printable via browser print layout.

---

## 7. Notes on Data Integrity

- Receipt content is derived from finalized sale snapshots and receipt snapshots.
- Receipt does not recompute totals, VAT, discount, or FX conversions.
- Any discrepancies must be resolved in Sale/Cash Session/Policy modules, not in Receipt.

---

## 8. Out of Scope (Future Work)

- Thermal printer integration and device pairing
- Digital receipt delivery (SMS/email)
- Receipt template designer
- Multiple languages or per-item tax breakdowns
- Custom fields like tax ID, business registration number (Capstone II candidate)

---

# End of Document

## Audit Events Emitted

The following events MUST be written to the Audit Log when triggered (with tenant_id, branch_id where applicable, actor_id, and relevant entity IDs):

- `RECEIPT_VIEWED`
- `RECEIPT_PRINT_REQUESTED`

