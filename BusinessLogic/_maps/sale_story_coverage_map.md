# Story Coverage Map — Order & Sale (Selling, Serving, Correcting)

**Purpose:**  
Identify human-centered stories for Order/Sale behavior and map them to technical use cases, ensuring complete coverage without “one story per UC”.

**Scope:**  
- Order Domain (`BusinessLogic/2_domain/40_POSOperation/order_domain_patched.md`)
- Sale Module Spec (`BusinessLogic/5_modSpec/30_POSOperation/sale_module_patched.md`)

**Rule:**  
Stories are grouped by **human operational context**, not by system modules.

---

## 1) Source Artifacts (Authoritative)

- **Domain:** `BusinessLogic/2_domain/40_POSOperation/order_domain_patched.md`
- **ModSpec:** `BusinessLogic/5_modSpec/30_POSOperation/sale_module_patched.md`

---

## 2) Operational Contexts (Human Reality)

From reading the domain and modspec, Order/Sale behavior is experienced by humans in **four distinct operational contexts**:

1. **Selling & Checkout**  
2. **Fulfilling Orders**  
3. **Correcting Mistakes**  
4. **Operating Under Constraints (Offline / Rules)**  

These contexts are stable and human-recognizable.

---

## 3) Story Set (Human Situations)

### Context: Selling & Checkout

1. **Completing a Sale at the Counter**  
   File: `BusinessLogic/1_stories/selling_and_checkout/completeing_a_sale.md`  
   Human situation: serving a customer, taking payment, committing the sale.

2. **Building and Reviewing an Order Before Checkout**  
   File: `BusinessLogic/1_stories/selling_and_checkout/review_order_pre-checkout.md`  
   Human situation: adjusting items, quantities, discounts, and payment method before committing.

---

### Context: Fulfilling Orders

3. **Preparing and Delivering an Order**  
   File: `BusinessLogic/1_stories/selling_and_checkout/prep_and_deliver_order.md`  
   Human situation: kitchen or staff preparing items and marking progress until delivery.

---

### Context: Correcting Mistakes

4. **Voiding a Sale When a Mistake Happens**  
   File: `BusinessLogic/1_stories/correcting_mistakes/voiding_a_sale.md`  
   Human situation: correcting a completed sale via request and approval.

---

### Context: Operating Under Constraints

5. **Completing a Sale When the System Is Offline**  
   File: `BusinessLogic/1_stories/selling_and_checkout/selling_while_offline.md`  
   Human situation: continuing to serve customers despite connectivity issues.

---

## 4) Coverage Mapping (Use Case → Story)

### Story 1 — Completing a Sale at the Counter
Covers:
- **UC-4:** Finalize Sale (Checkout)
- **UC-5:** View Orders & eReceipt (immediate post-sale review)

Notes:
- This is the *happy path*.
- Emphasizes finality and trust in recorded totals.

---

### Story 2 — Building and Reviewing an Order Before Checkout
Covers:
- **UC-1:** Access Sale Screen
- **UC-2:** Start Draft Order (Add Item)
- **UC-3:** Cart / Pre-checkout (Edit + Compute)

Notes:
- Draft behavior is device-local and exploratory.
- No financial commitment yet.

---

### Story 3 — Preparing and Delivering an Order
Covers:
- **UC-6:** Update Order Fulfillment Status

Notes:
- Fulfillment is operational, not financial.
- Order status changes must never affect sale totals.

---

### Story 4 — Voiding a Sale When a Mistake Happens
Covers:
- **UC-7:** Request Void (Cashier)
- **UC-8:** Approve & Execute Void (Manager/Admin)

Notes:
- Treated as an exceptional workflow.
- Reversal occurs only after approval.

---

### Story 5 — Completing a Sale When the System Is Offline
Covers:
- **UC-9:** Offline Finalize & Sync

Notes:
- Focus on trust: “the system will reconcile later without duplicating.”

---

## 5) Coverage Summary (Quick Check)

| Use Case | Covered By Story |
|---|---|
| UC-1 Access Sale Screen | Story 2 |
| UC-2 Start Draft Order | Story 2 |
| UC-3 Cart / Pre-checkout | Story 2 |
| UC-4 Finalize Sale | Story 1 |
| UC-5 View Orders & eReceipt | Story 1 |
| UC-6 Update Fulfillment Status | Story 3 |
| UC-7 Request Void | Story 4 |
| UC-8 Approve / Reject Void | Story 4 |
| UC-9 Offline Finalize & Sync | Story 5 |

---

## 6) Status Tracker (Fill As You Write)

| Story | File | Status | Notes |
|---|---|---|---|
| Completing a Sale at the Counter | `BusinessLogic/1_stories/selling_and_checkout/completeing_a_sale.md` | ok | |
| Building and Reviewing an Order Before Checkout | `BusinessLogic/1_stories/selling_and_checkout/review_order_pre-checkout.md` | ok | |
| Preparing and Delivering an Order | `BusinessLogic/1_stories/selling_and_checkout/prep_and_deliver_order.md` | ⬜ | |
| Voiding a Sale When a Mistake Happens | `BusinessLogic/1_stories/correcting_mistakes/voiding_a_sale.md` | ok | |
| Completing a Sale When Offline | `BusinessLogic/1_stories/selling_and_checkout/selling_while_offline.md` | ok | |

Legend:
- ⬜ Not written
- 🟨 Drafted
- ✅ Final

---

## 7) Guardrails (Important)

- Do **not** split stories per UC.
- Do **not** duplicate stories across folders.
- Stories must remain non-technical and human-centered.
- If a new UC is added later:
  - first try to fit it into an existing story,
  - only add a new story if it represents a new human situation.

---

# End of Map
