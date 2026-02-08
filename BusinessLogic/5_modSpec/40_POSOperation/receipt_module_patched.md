# Receipt Module — Feature Module Spec (Capstone 1)

**Version:** 1.1 (Patched)  
**Status:** Clarified to align with snapshot-based Sale lifecycle and cross-module processes.  
**Module Type:** Feature Module  
**Primary Domain:** POS Operations → Receipt  
**Depends on:** Sale/Order, Tenant & Branch Context, Audit Logging  
**Collaborates with (Cross-Module):** Sale/Order (Finalize Sale, Void Sale), Reporting

---

## 1. Purpose

The Receipt module is responsible for **rendering and presenting immutable receipt snapshots** for finalized sales.

A receipt:
- reflects the **exact state of a sale at finalization time**
- is used for customer-facing display, printing, and sharing
- does **not** compute prices, taxes, discounts, or totals

**Important boundary:**
- Receipt does **not** mutate business state
- Receipt does **not** re-compute or re-interpret data
- Receipt renders frozen truth produced by the Sale/Finalize process

---

## 2. Receipt Lifecycle (Authoritative)

- A receipt snapshot is created **exactly once**, at **Sale Finalization**
- Receipt data is **immutable** after creation
- Viewing, printing, or re-printing a receipt does **not** change:
  - sale state
  - cash state
  - inventory state

Receipts are always associated with a **finalized Sale**, never with draft orders.

---

## 3. Scope

### Includes
- Generating a receipt snapshot at sale finalization
- Rendering receipt data for:
  - on-screen viewing
  - printing
  - digital sharing (e.g., PDF / image)
- Displaying receipt state (NORMAL, VOID_PENDING, VOIDED)
- Displaying identity snapshots:
  - tenant name
  - branch name
  - footer / legal text captured at finalization time

### Excludes
- Price calculation
- Discount calculation
- Inventory deduction
- Cash movement
- Hardware integration (printer drivers, USB/Bluetooth handling)

---

## 4. Data Ownership

Receipt owns a **presentation snapshot**, which may include:
- receipt_id
- sale_id (foreign reference)
- receipt_number / display_number
- sale timestamp
- line items (copied from sale snapshot)
- applied discounts (copied from sale snapshot)
- totals (copied from sale snapshot)
- tenant & branch identity snapshot
- footer text snapshot

Receipt data must never diverge from the finalized Sale snapshot.

---

## 5. Use Cases — Self-Contained Processes

### UC-1: View Receipt
**Actors:** Admin, Manager, Cashier  
**Preconditions:** Sale is finalized  
**Flow:** Retrieve receipt snapshot by receipt_id or sale_id  
**Postconditions:** None (read-only)

---

### UC-2: Print Receipt
**Actors:** Admin, Manager, Cashier  
**Preconditions:** Receipt exists  
**Flow:** Send receipt snapshot to printing layer  
**Postconditions:** None (printing is observational)

---

### UC-3: Reprint Receipt
**Actors:** Admin, Manager  
**Rules:** Reprint uses the same immutable snapshot  
**Postconditions:** None

---

### UC-4: View Void State on Receipt
**Actors:** Admin, Manager, Cashier  
**Flow:** Display receipt with:
- VOID_PENDING watermark when reversal in progress
- VOIDED watermark after reversal completed  
**Postconditions:** None

---

## 6. Cross-Module Interaction (Read-Only)

### CM-1: Receipt Creation (Triggered by Finalize Sale)
**Trigger:** Finalize Sale process  
**Behavior:**
- Sale module produces a finalized sale snapshot
- Receipt module persists a receipt snapshot derived from that sale

Receipt creation must be idempotent per sale_id.

---

### CM-2: Receipt Display After Void
**Trigger:** Sale voided  
**Behavior:**
- Receipt remains immutable
- Visual state reflects sale status (VOID_PENDING / VOIDED)
- No recomputation of amounts

---

## 7. Idempotency & Offline Safety

- Receipt creation is idempotent per sale_id
- Replaying Finalize Sale must not create duplicate receipts
- Viewing/printing is always safe under retry

---

## 8. Audit & Observability

The following events may be recorded:

- `RECEIPT_CREATED`
- `RECEIPT_VIEWED`
- `RECEIPT_PRINT_REQUESTED`

**Clarification:**
- Receipt audit events are **observational**
- They do **not** represent business-state changes
- They must not be used to trigger financial or inventory logic

---

## 9. Out of Scope (Capstone 1)

- Custom receipt templates
- Multi-language receipts
- Advanced printer configuration
- Digital receipt delivery (SMS, email)
- Refund receipts (separate process)

---

## 10. Design Notes

Receipt is intentionally simple.

By treating receipts as immutable snapshots:
- disputes are easier to resolve
- reporting remains consistent
- offline replay is safe
- future formatting changes do not affect history

Receipt exists to **show truth**, not to decide it.

---

# End of Document
